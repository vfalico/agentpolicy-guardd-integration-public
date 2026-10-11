# Install and test

This guide is for a disposable Linux/Kubernetes test environment. Review and
adapt paths, identities, storage, and key management before any production use.

## 1. Clone the exact integration sources

```bash
branch=agentpolicy-kmx-orka-ig-guardd-current

git clone --branch "$branch" https://github.com/vfalico/kaimahi-public.git
git clone --branch "$branch" https://github.com/vfalico/orka-public.git
git clone --branch "$branch" https://github.com/vfalico/ig-kmx-guardd-public.git
git clone https://github.com/vfalico/inspektor-gadget-public.git

git -C kaimahi-public checkout \
  fddda19046bff9b97244c4fbde2b01d8123b086a
git -C orka-public checkout \
  95effe76f9378b991c97cb86a3d753634afe9004
git -C ig-kmx-guardd-public checkout \
  f0df7157d164959e21831218475b5e174814ed72
git -C inspektor-gadget-public checkout \
  b6dce71343f02269051fdea4e0d463d72c1d5310
```

Record every checked-out commit before building:

```bash
for repo in kaimahi-public orka-public ig-kmx-guardd-public \
  inspektor-gadget-public; do
  git -C "$repo" rev-parse HEAD
  git -C "$repo" status --short
done
```

All worktrees must be clean. Do not substitute upstream Inspektor Gadget source
or change canonical Go module paths.

## 2. Build and test Kaimahi

Use the repository's documented Go version:

```bash
cd kaimahi-public
go test ./internal/kmx/policy
go test ./cmd/kmx -run '^TestTheCommandTreeIsExactlyWhatIsListedHere$'
go build -o ./bin/kmx ./cmd/kmx
cd ..
```

If the full `./cmd/kmx` package is tested from a shared-group checkout, ensure
the repository files have their Git-declared non-group-writable modes first.
The suite builder deliberately rejects writable source content.

Create a policy and binding using the schema described in
`kaimahi-public/docs/agentsuite-policy.md`, then render them without contacting
a cluster:

```bash
kaimahi-public/bin/kmx suite render-policy \
  policy.json binding.json --namespace agents > policy.yaml
```

The command must reject unknown fields and unsupported network capabilities.

## 3. Build guardd and the gadget

Follow `ig-kmx-guardd-public/README.md` and
`ig-kmx-guardd-public/gadget/README.md`.

At minimum:

```bash
cd ig-kmx-guardd-public
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -e .
.venv/bin/python -m unittest discover -s tests -v

cd gadget
go work init . ../../inspektor-gadget-public
go work edit \
  -replace=github.com/inspektor-gadget/inspektor-gadget@v0.0.0=../../inspektor-gadget-public
go test ./...
go vet ./...
rm go.work
cd ../..
```

Build the owner-fork `ig` executable and OCI gadget archive as described by the
gadget guide. Record SHA-256 digests for both artifacts. The digest configured
in guardd and Orka must identify the same gadget.

## 4. Generate keys

Use distinct Ed25519 key pairs for authorization and receipts:

```bash
openssl genpkey -algorithm ED25519 -out authorization-private.pem
openssl pkey -in authorization-private.pem -pubout \
  -out authorization-public.pem
openssl genpkey -algorithm ED25519 -out receipt-private.pem
openssl pkey -in receipt-private.pem -pubout -out receipt-public.pem
chmod 0600 authorization-private.pem receipt-private.pem
chmod 0644 authorization-public.pem receipt-public.pem
```

guardd receives the authorization public key and receipt private/public pair.
Orka receives the authorization private key and receipt public key. Do not
place the guardd receipt private key in Kubernetes.

## 5. Install guardd on every test node

Install the Python package, owner-fork `ig` binary, gadget OCI archive, key
material, service unit, and a completed guardd configuration. Verify every
artifact digest before starting the service.

Install guardd into the path pinned by the packaged unit:

```bash
cd ig-kmx-guardd-public
sudo python3 -m venv /opt/ig-kmx-guardd/venv
sudo /opt/ig-kmx-guardd/venv/bin/python -m pip install --upgrade pip
sudo /opt/ig-kmx-guardd/venv/bin/python -m pip install .
```

The configured authorization issuer ID, gadget digest, node ID, socket path,
and cgroup root must match the Orka configuration. The socket and receipt
directories must be dedicated to this integration. The example
`allowed_uids: [0]` matches the Orka supervisor container's Unix peer UID; if
that security context changes, use the exact replacement UID. COSE
authorization remains mandatory.

Start guardd and verify:

```bash
sudo systemctl start ig-kmx-guardd.service
sudo systemctl status --no-pager ig-kmx-guardd.service
sudo test -S /run/ig-kmx-guardd/guardd.sock
```

## 6. Build and install Orka

Build and test the changed policy packages:

```bash
cd orka-public
make --quiet ensure-ui-embed
go test ./api/v1alpha1 ./internal/policy ./internal/guarddclient \
  ./internal/harness/v2 ./workers/acp/supervisor
go test ./internal/controller -run \
  '^(TestKubernetesTaskPolicyGateValidatesSourcesBeforeRuntimeActivation|TestRuntimePoolAgentPolicyConfigRejectsPartialConfiguration|TestRuntimePoolPodTemplateWiresNodeLocalGuarddBoundary|TestRuntimePoolPodTemplateWithoutAgentPolicyRemainsRestricted|TestTaskPolicyActivationStatusProjectsVerifiedAttachmentsAsArray|TestDeletingACPTaskRetiredBeforeDurableAttemptUsesNoAttemptReclamation|TestDeletingACPTaskSettlesBoundPlannedAttemptBeforeReclamation|TestDeletingACPTaskRetiredBeforeDispatchRejectsContradictoryEvidence|TestDeletingACPTaskRetiredBeforeDispatchUsesAuthoritativeContradictionReader|TestDeletingACPTaskRetiredBeforeDispatchCASPreservesValidatedDeliveryState)$'
go build ./cmd/...
rm -rf internal/uiembed/dist
cd ..
```

The full controller suite additionally requires the repository's pinned
Kubebuilder `etcd` and API-server assets. The focused command above covers the
changed policy surfaces without pretending those external assets are present.

Install the CRDs and chart from the integration branch. Configure the complete
`controller.acpRuntime.agentPolicy` tuple:

- `guarddSocketHostPath`;
- `authorizationIssuerID`;
- `authorizationPrivateKeyHostPath`;
- `receiptPublicKeyHostPath`;
- `receiptDirectoryHostPath`;
- `gadgetDigest`; and
- `cgroupRoot`.

Partial configuration is intentionally rejected. The runtime namespace must be
dedicated, and the receipt directory must be root-owned mode `0700`.

## 7. Apply a policy and create a bound Task

Apply the rendered `AgentPolicy` and `AgentPolicyBinding`. Set the Task's
immutable `spec.policyBindingRef` to the binding name. Confirm that the binding
reports ready before creating the Task.

Observe the Task and runtime:

```bash
kubectl -n agents get agentpolicy,agentpolicybinding,task
kubectl -n agents get task <task-name> -o yaml
```

Do not treat resource creation as proof of enforcement. The Task must show a
current policy activation with a verified receipt and attachment entries before
prompt dispatch.

## 8. Manual acceptance

Run each check with a fresh Task and exact evidence capture:

1. **Held startup:** delay or refuse guardd activation and prove the provider
   does not execute.
2. **Exact write denial:** deny one path, attempt a write to it, and require
   `EPERM`; write to a neighboring unlisted path and require success.
3. **Attachment evidence:** verify the signed receipt names the expected gadget
   and observed BPF programs.
4. **Fail-safe faults:** independently terminate the IG worker, event consumer,
   guardd, and guardian control channel; require the target cgroup to freeze
   before retirement.
5. **Restart:** restart controller and guardd processes separately and verify
   durable state reconciles without releasing an unverified workload.
6. **Concurrent isolation:** run inverse policies concurrently and prove each
   cgroup receives only its own decision.
7. **Unsupported capability:** request network policy and require rejection
   before activation.
8. **Cleanup:** require zero test cgroups, guardd workers, gadget processes,
   AgentPolicy Tasks, and AgentPolicy BPF programs after teardown.

For exact executable enforcement, use the direct guardd/gadget policy boundary.
The portable Kaimahi-to-Orka contract does not author executable rules.

## Expected result

A successful run proves the tested policy decision and its activation chain for
the exact commits and artifacts recorded for that run. It does not prove the
non-claims listed in the repository README or security model.

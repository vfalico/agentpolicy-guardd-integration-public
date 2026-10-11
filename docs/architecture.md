# Architecture

## The problem

An agent policy is useful only if the system can prove that the intended
policy, runtime, workload, and kernel attachment all refer to the same
execution. Starting an agent and attaching enforcement afterward creates a
race. Treating an event stream as enforcement also fails because an event is
observed after the kernel has already decided.

This integration closes those gaps with two release gates and explicit
identity binding.

## Components

### Kaimahi and KMX

Kaimahi is the policy-authoring boundary. `kmx suite render-policy` validates a
strict JSON contract and emits namespaced Orka `AgentPolicy` and
`AgentPolicyBinding` resources. It rejects unknown fields, unsupported
capabilities, duplicate rules, invalid paths, invalid time windows, and source
UID or digest mismatches.

The portable contract is intentionally narrow: exact-path write denial. A
capability that cannot be represented and enforced exactly is rejected before
rendering.

### Orka controllers

Orka stores the policy and immutable binding. Before dispatching a Task, the
controller re-resolves every source through an uncached API reader and checks
UIDs, canonical digests, capabilities, validity windows, subject identity, and
Task identity.

The controller does not treat API admission as kernel enforcement. It waits for
durable activation status containing a current signed receipt and attachment
identity.

### ACP supervisor

The supervisor is the workload-start boundary. It:

1. creates one exact child cgroup;
2. starts the provider process stopped;
3. moves the process into that cgroup;
4. opens the cgroup directory;
5. sends the descriptor to guardd with `SCM_RIGHTS`;
6. verifies the signed guardd receipt;
7. persists the receipt before release; and
8. releases the provider only after all checks pass.

The supervisor is unprivileged relative to BPF loading. It holds a request
authorization key and a receipt verification key, not the guardd receipt
signing key.

### guardd

guardd is the sole privileged Inspektor Gadget actuator. It validates the peer,
COSE authorization, request binding, exact cgroup descriptor, policy digest,
gadget digest, node identity, and durable lifecycle state.

It loads the pinned gadget, confirms the worker and map state, correlates the
kernel objects, arms the fail-safe guardian, and issues a hash-chained signed
receipt. Incomplete activation never produces a release-eligible receipt.

### Inspektor Gadget and BPF LSM

The upstream Inspektor Gadget checkout builds and loads the AgentPolicy gadget
while preserving canonical Go import paths. The gadget compiles the strict
policy into bounded BPF maps and switches the policy state to active only after
map population completes.

BPF LSM hooks synchronously return allow or deny. Denial events describe a
decision already made by the kernel; they are not an asynchronous policy
engine.

## One governed execution

```text
policy JSON + binding JSON
    |
    | strict validation and canonical digest
    v
AgentPolicy + AgentPolicyBinding
    |
    | immutable source and subject identity
    v
Task.spec.policyBindingRef
    |
    | controller resolution and admission
    v
held provider process in exact child cgroup
    |
    | cgroup directory FD + signed activation request
    v
guardd -> IG -> gadget -> BPF LSM
    |
    | verified guardian-bound receipt
    v
provider release
    |
    | durable PolicyReady activation projection
    v
prompt dispatch
```

## The two release gates

The gates answer different questions:

1. **Supervisor release gate:** is the stopped provider in the exact cgroup with
   a verified, guardian-bound kernel policy attachment?
2. **Controller dispatch gate:** has that activation been validated and
   durably projected for this exact Task generation?

Passing the first gate does not bypass the second. This separation prevents a
valid low-level receipt from being used for the wrong Task or from disappearing
before control-plane dispatch.

## Identity flow

The binding pins suite, agent, composition, sandbox, policy UID, and canonical
policy digest. The activation request adds session, attempt, generation, gadget
digest, and node identity. The received cgroup descriptor is resolved to a
kernel cgroup identity. The receipt binds those values with guardian readiness
and observed gadget/BPF attachment names.

Every transition rejects drift rather than guessing which identity is current.

## Failure and reconciliation

Before release, any missing configuration, invalid signature, digest mismatch,
dead worker, failed attachment, incomplete receipt, or persistence failure
retires the attempt without running the provider.

After activation, the guardian monitors the IG worker, event consumer, guardd
parent, and its control channel. Loss of a required component freezes the
target cgroup before retirement. Orka and guardd both persist lifecycle state
so restart recovery can reconcile rather than silently recreate authority.

The accepted scope does not include a second actor that automatically freezes a
workload after the guardian process itself dies.

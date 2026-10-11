# AgentPolicy, KMX, Orka, guardd, and Inspektor Gadget integration

This repository explains a fail-closed path from a portable agent policy to a
kernel-enforced workload boundary:

```text
Kaimahi / KMX
    |
    | renders AgentPolicy + AgentPolicyBinding
    v
Orka controllers
    |
    | resolves immutable policy identity and gates Task dispatch
    v
ACP supervisor
    |
    | starts the provider stopped and passes an exact cgroup FD
    v
ig-kmx-guardd
    |
    | loads a digest-pinned gadget and returns signed attachment evidence
    v
Inspektor Gadget + BPF LSM
    |
    | synchronously allows or denies the kernel operation
    v
one cgroup-v2 workload
```

The current portable contract supports exact-path write denial. The lower
guardd/gadget layer also supports exact executable denial, but the portable
Kaimahi-to-Orka API does not author executable rules in this release. Network
policy is rejected rather than approximated.

## Repositories

Use the branch `agentpolicy-kmx-orka-ig-guardd-current` unless a later release
names a different integration branch.

The immutable source identities for this candidate are recorded in
[`release-manifest.json`](release-manifest.json). Clone the branch for
discoverability, then check out the exact commit before building.

| Repository | Role |
|---|---|
| `vfalico/kaimahi-public` | validates and renders the portable policy and binding |
| `vfalico/orka-public` | resolves policy identity, holds provider startup, verifies receipts, and gates prompt dispatch |
| `vfalico/ig-kmx-guardd-public` | privileged node service, fail-safe guardian, signed evidence, and BPF LSM gadget |
| `vfalico/inspektor-gadget-public` | owner-controlled IG build with canonical Go module paths |
| `vfalico/agentpolicy-guardd-integration-public` | architecture, installation, testing, and reproducibility guide |

Start with:

- [Architecture](docs/architecture.md)
- [Install and test](docs/install-and-test.md)
- [Security model](docs/security-model.md)
- [Reproducibility](docs/reproducibility.md)

## What this demonstrates

- policy identity is pinned by immutable UIDs and canonical digests;
- provider execution is held until guardd activation is independently
  verified;
- guardd receives an exact cgroup directory descriptor rather than trusting a
  path;
- signed receipts bind policy, runtime, cgroup, guardian, gadget, and kernel
  attachment identities;
- BPF LSM hooks make the decision before the syscall continues;
- loss of required control processes triggers a fail-safe cgroup freeze; and
- Orka persists validated activation before prompt dispatch.

## What this does not claim

This is not a general sandbox, network firewall, or complete file-access
confinement system. It does not claim read confinement, rename/unlink
mediation, inherited-descriptor coverage, hard-link coverage, `io_uring`
coverage, or guardian-process-death recovery by a second independent actor.
Receipts prove verified activation facts; deny/allow behavior must still be
tested at the kernel boundary.

## License

Documentation in this repository is licensed under Apache-2.0.

# Security model

## Protected properties

The integration is designed to preserve these properties:

- no provider execution before verified policy activation;
- no path-string substitution for cgroup authority;
- no acceptance of a policy, gadget, node, session, or Task identity that
  differs from the signed binding;
- no release receipt before worker, maps, cgroup, attachment, and guardian
  checks complete;
- no prompt dispatch before validated activation is durable; and
- fail-safe freezing when a required post-activation control process is lost.

## Principals and keys

The Orka supervisor is authorized to request activation with an Ed25519 private
key. guardd holds the allowlisted public key. guardd signs lifecycle receipts
with a separate private key; Orka holds only its public verification key.

Peer credentials and the configured UID allowlist are checked at the Unix
socket. Cryptographic authorization is still required; peer UID alone is not
sufficient.

## Cgroup authority

The caller opens the cgroup directory and transfers that descriptor with
`SCM_RIGHTS`. guardd validates the descriptor and derives the cgroup identity
from it. A caller-provided path is not reopened and does not become authority.

The workload is placed in an exact child cgroup before activation. Policies are
keyed to that cgroup identity, which provides isolation between concurrent
workloads with inverse rules.

## Policy activation

Policy JSON is strict, bounded, and canonicalized. The gadget populates bounded
maps before changing policy state to active. Startup behavior is fail closed
for the target cgroup.

Executable and file keys are fixed-width map keys. Kernel-resolved paths are
normalized and their unused key bytes are zeroed so visually identical paths
cannot differ because of stale helper-buffer contents.

## Receipts and evidence

Receipts are canonical CBOR wrapped in COSE Sign1/Ed25519 and linked by prior
receipt hash. The release-eligible receipt binds:

- policy and gadget digests;
- session, attempt, and generation;
- node and exact cgroup identity;
- guardian readiness;
- IG worker identity; and
- observed gadget and BPF program identities.

A receipt proves that those activation checks completed. It does not by itself
prove every later syscall decision. Behavioral acceptance must exercise the
kernel deny and neighboring allow cases.

## Fail-safe guardian

The guardian holds exact process/file descriptors for the IG worker, event
consumer, guardd parent, control channel, and target cgroup. When an accepted
fault is detected, it writes the cgroup freezer directly and records bounded
terminal evidence.

Accepted automatic freeze cases are worker death, event-consumer loss, guardd
death, and control-channel loss while the guardian remains alive. Guardian
process death itself is outside the accepted automatic-freeze scope because
there is no second independent guardian.

## Privilege boundary

guardd requires privileges to load BPF programs, inspect kernel objects, and
operate the cgroup freezer. It should run only on dedicated or carefully
controlled nodes. The packaged unit narrows capabilities and writable paths,
but systemd hardening is defense in depth, not a replacement for host trust.

Kubernetes workloads receive only the exact socket, authorization private key,
receipt public key, receipt directory, and cgroup tree required by the
integration. The guardd receipt private key and unrestricted host filesystem
are not mounted into the workload.

## Explicit non-claims

This release does not claim:

- general read confinement;
- complete create/replace/rename/unlink/hard-link mediation;
- inherited writable-descriptor or `io_uring` coverage;
- network enforcement;
- policy-derived process termination;
- immunity to a compromised privileged host or guardd process; or
- automatic freeze after guardian-process death.

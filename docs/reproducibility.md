# Reproducibility and source provenance

## Source boundary

The integration uses canonical Go module and import paths. Source selection is
separate from import identity:

- Orka and Kaimahi retain their existing module paths;
- the gadget retains
  `github.com/inspektor-gadget/inspektor-gadget` imports; and
- a local Go workspace or temporary vendor tree selects the upstream
  Inspektor Gadget checkout.

Do not rewrite canonical module paths. Acceptance used upstream commit
`b6dce71343f02269051fdea4e0d463d72c1d5310`; current upstream IG may also be
used.

## Record an exact build

For every repository, record:

```bash
git rev-parse HEAD
git rev-parse HEAD^{tree}
git status --short
git submodule status --recursive
```

Also record:

- Go and Python versions;
- operating-system and kernel versions;
- BPF LSM and cgroup-v2 availability;
- the IG executable SHA-256;
- the gadget OCI archive SHA-256;
- the gadget manifest digest configured in guardd and Orka;
- rendered Kubernetes manifest digests; and
- the policy and binding canonical digests.

The worktrees must be clean. A dirty build is not the same candidate.

## Local IG selection

For Go host tests, create a local workspace:

```bash
go work init ./ig-kmx-guardd-public/gadget ./inspektor-gadget
go work edit \
  -replace=github.com/inspektor-gadget/inspektor-gadget@v0.0.0=./inspektor-gadget
```

For an image builder that cannot see the parent workspace, use a temporary
`replace` and vendor tree exactly as documented in the gadget README. Remove
them afterward and require `git diff --exit-code`.

## Generated surfaces

Orka includes generated deep-copy code, CRDs, Helm chart CRDs, and rendered
deployment manifests. A candidate is complete only when source API changes and
all generated surfaces agree. Regeneration must not leave an unexplained diff.

## Artifact identity

The same gadget identity must cross every layer:

1. OCI archive bytes and SHA-256;
2. OCI manifest digest;
3. guardd configuration;
4. Orka controller/runtime configuration;
5. activation request;
6. signed receipt; and
7. persisted Task activation status.

Any mismatch is a failed build or failed activation, not a warning.

## Verification limits

Matching Git trees proves source equality, not behavioral correctness. Matching
artifact digests proves byte identity, not that the artifact was built from the
claimed source. Preserve both source provenance and independent behavioral
tests.

The recommended evidence set is:

- clean source commits and trees;
- toolchain and generated-surface checks;
- artifact hashes;
- signed activation receipts;
- exact denied and neighboring allowed operations;
- fault-attribution evidence; and
- deterministic cleanup evidence.

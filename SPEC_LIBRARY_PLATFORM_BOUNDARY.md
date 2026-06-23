# ADO Spec Library And Platform Boundary

ADO has two repositories with different authority, release cadence, and change
control. They must not be treated as one repository with two folders.

```text
/Users/iminhyeog/dev/agent/
  ADO/                  # ADO Spec Library
  ado-platform/         # Public ADO implementation monorepo
  projects/             # product repositories managed by ADO
```

The path names are local defaults. Git remotes, immutable commits, and approved
revision records are authoritative; an absolute local path is never part of an
agent packet or a durable Project contract.

## 1. Authority Split

| Repository | Owns | Must not own |
|---|---|---|
| `ADO` Spec Library | constitutional rules, templates, schemas, architecture contracts, security and state constraints | application runtime, database migrations, provider credentials, project worktrees |
| `ado-platform` | public Nest API, Worker, Control UI, database migrations, runtime adapters, CI, deployment-local configuration | unilateral redefinition of ADO constitutional rules |
| managed Project repository | product code, project-specific constraints, roadmap-derived documents, implementation PRs | mutation of the pinned ADO Spec revision |

The Spec Library defines what ADO is permitted to do. The Platform implements
those rules. A managed Project is a target of the Platform and has its own
project-specific constraints under the precedence rules in `ADO_MASTER_SPEC.md`.

Both ADO repositories are public in v1 so GitHub Free branch protection can
enforce the PR-only model. Public visibility never permits secrets, raw logs,
unredacted artifacts, private environment files, provider credentials, or
internal database exports to enter either repository.

## 2. Spec Library Repository Rules

The `ADO` directory must become a protected Git repository before Platform A1
begins. It has a protected default branch and uses human-reviewed PRs for every
canonical change. A specification revision is an immutable Git commit on an
approved protected branch or signed release tag.

The Spec Library publishes a deterministic manifest containing at least:

```text
spec_schema_version
spec_revision                 # full immutable commit SHA
approved_ref                  # protected branch or release tag name
manifest_sha256
canonical_document_hashes     # path -> SHA-256
template_hashes               # path -> SHA-256
schema_hashes                 # path -> SHA-256
generated_at
```

The manifest is an Artifact when imported into the Platform. It never contains
secrets, a mutable local path, or unreviewed working-tree content.

`manifest_sha256` is the SHA-256 of the canonical UTF-8 JSON payload with the
`manifestSha256` field omitted. This avoids a self-referential file hash while
still binding every declared document/template/schema hash and manifest field.

An ordinary edit to a checked-out `ADO/` file is not a usable specification
revision. The Platform may use it only after the change is committed, reviewed,
listed in the manifest, and imported as an approved SpecLibraryRevision.

## 3. Platform Pinning

`ado-platform` contains an `ado-spec.lock.json` at its repository root. The
file is committed with Platform code and pins exactly one approved Spec Library
revision for a Platform release/branch.

```json
{
  "lockSchemaVersion": 1,
  "specRepository": "configured-canonical-remote",
  "specRevision": "full-immutable-git-commit-sha",
  "approvedRef": "protected-branch-or-signed-tag",
  "manifestSha256": "sha256-of-imported-manifest",
  "canonicalDocumentHashes": {
    "ADO_MASTER_SPEC.md": "sha256...",
    "NESTJS_MONOREPO_ARCHITECTURE.md": "sha256..."
  }
}
```

The lock is not an editable agent input. CI verifies that it resolves to an
approved immutable revision and that the listed hashes equal the imported
manifest. A Platform PR that changes the lock must state why the rules changed,
show its compatibility assessment, and receive Human Owner approval.

`schemas/spec_library_manifest.schema.json` validates the published manifest;
`schemas/ado_spec_lock.schema.json` validates the Platform lock. Schema-valid
content is still rejected when its commit, hashes, or approval state disagree
with the imported SpecLibraryRevision record.

The Platform may cache a read-only copy of the locked specification in CI or
build artifacts for validation. It must not fork, silently patch, or use an
uncommitted specification checkout at runtime.

## 4. Database And Runtime Lineage

The Platform imports approved manifests into the append-only
`core.SpecLibraryRevision` record. Its minimum fields are:

| Field | Rule |
|---|---|
| `revision_key` | stable human-readable key, unique |
| `repository_identity` | configured remote identity, not a local path |
| `commit_sha` | full immutable Git SHA, unique with repository identity |
| `approved_ref` | protected branch/tag used for approval |
| `manifest_sha256` | integrity anchor for the imported document set |
| `manifest_artifact_id` | immutable imported manifest Artifact |
| `status` | `approved`, `deprecated`, or `revoked`; never mutable history |
| `approved_by_actor_id`, `approved_at` | Human Owner approval evidence |

Every Project has one active `spec_library_revision_id`. Changing it creates a
new Project configuration revision through an approved transition; it does not
rewrite prior work. Every ContextPacket, ReviewPacket, AgentRun, VerificationRun,
and PullRequestPacket records its effective SpecLibraryRevision and manifest
hash, even when it inherits the Project default.

The Worker refuses to execute an Agent Run when the packet's revision is absent,
revoked, hash-mismatched, or not compatible with the current Platform lock.
This is a preflight failure with evidence, not a best-effort warning.

## 5. Controlled Change Flow

```text
Spec Library PR
-> human approval and immutable commit/tag
-> manifest publication
-> Platform lock-update PR with compatibility assessment
-> human approval
-> platform deployment/release
-> Project-level adoption request where required
-> new packets/runs use the new effective revision
```

Existing execution records remain attached to their original revision. A
revision may be deprecated for new work or revoked for a security defect.
Revocation pauses affected new execution and creates a SafetyEvent/incident;
it never changes prior historical records to claim that they used another rule.

## 6. Context Efficiency Rules

Agents do not receive the whole Spec Library by default. A ContextPacket
contains only:

- the effective `spec_revision` and `manifest_sha256`;
- the named canonical sections required by the assigned role;
- Project, Feature Unit, Component Work, and verification constraints needed
  for the current task;
- a bounded artifact/source list with hashes.

The packet records every selected document and hash. The agent may not replace
that selection with a mutable checkout, a newer branch tip, or a document from
another Project.

## 7. Bootstrap Consequences

Before A1 implementation begins, B0 must initialize/protect the Spec Library
Git repository, publish its first approved manifest, create the empty
`ado-platform` Git repository, protect `main`, create `integrate`, and commit
an initial `ado-spec.lock.json` that pins the first Spec Library revision.

A1 runs only in `ado-platform`. The Spec Library's documents remain the source
of design constraints and are changed only through the controlled change flow.

## 8. Acceptance Criteria

This boundary is established only when:

1. `ADO` and `ado-platform` are separate Git repositories with independent PR
   histories and protected branch rules;
2. Platform CI rejects a missing, mutable, hash-mismatched, or unapproved Spec
   Library lock;
3. an imported `SpecLibraryRevision` gives every generated packet/run a
   verifiable effective rule revision;
4. an Agent Run cannot start from an unpinned or revoked revision;
5. Spec Library changes cannot be merged as incidental Platform code changes.

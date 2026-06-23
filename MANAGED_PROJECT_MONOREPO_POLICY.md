# Managed Project Monorepo Policy

Every product Project managed by ADO uses exactly one public Git monorepo in
v1. This policy applies to managed Projects, not the separate `ado-platform`
implementation repository. A future multi-repository exception requires an
explicit revision of this policy and is not silently enabled per Project.

## 1. Repository Model

One ADO Project has one active public Repository. App, web, API, backend
workers, admin surfaces, shared contracts, UI packages, infrastructure code,
and documentation live beneath that repository's declared component roots.

```text
project-repo/
  apps/
    mobile/
    web/
    admin/
  services/
    api/
    worker/
  packages/
    contracts/
    ui/
    shared/
  infra/
  docs/
```

The exact topology is Project-specific. ADO records each Component's
`ComponentRepository.relative_root`; it does not infer ownership from directory
names. Several Components may map to the same Repository, but their roots must
be explicit and may not ambiguously overlap without Human Owner approval.

## 2. Component Work Execution Scope

`ComponentWork` remains the branch, worktree, verification, and PR unit. It
has one repository, one `base_branch`, one `head_branch`, and one PR at most.

| Scope | When used | Git result |
|---|---|---|
| `single` | one Component can implement and verify the change independently | one branch and one PR for its one declared root |
| `coordinated` | acceptance requires atomic changes in two or more Components | one branch and one PR covering all declared component roots |

A coordinated Component Work is required for an API contract plus its server
and client consumers, shared package changes consumed by the same Feature Unit,
cross-component migrations, and any change whose verification cannot pass in a
partially merged state. It is not a collection of separate PRs waiting to be
merged in the correct order.

Per-component requirements remain visible as ComponentWorkScope entries and
spec sections. They may be implemented in parallel only inside the same
worktree when their allowed paths do not overlap. ADO does not create multiple
simultaneously writable worktrees for the same coordinated branch.

## 3. Branch And PR Rules

Every Component Work branch uses the work key, not a component key:

```text
ado/{project_key}/{work_type}/{feature_unit_key}/{component_work_key}
```

Examples:

```text
ado/deci-duel/feature/nickname-change/nickname-contract
ado/deci-duel/fix/profile-screen/profile-ui
```

Branches start from `integrate`; every ADO PR targets `integrate`; ADO never
pushes directly to `integrate` or targets/modifies `main`. Human review merges
only after the complete Feature Unit evidence is acceptable.

One Feature Unit may have several independent Component Works and PRs. When its
components must be released or validated together, it uses one coordinated
Component Work instead. The planner must state this choice and its reason
before human approval.

## 4. ComponentWorkScope Contract

Each Component Work has one primary Component and one or more immutable scope
entries:

| Field | Meaning |
|---|---|
| `component_id` | Component participating in this Work |
| `repository_id` | Must equal the Component Work repository |
| `relative_root` | Normalized root derived from approved ComponentRepository mapping |
| `scope_role` | `primary`, `contributing`, or `shared_contract` |
| `is_required` | Whether omission blocks Feature Unit acceptance |

Rules:

- `single` has exactly one `primary` entry and no other scope entry.
- `coordinated` has one `primary` entry and at least one `contributing` or
  `shared_contract` entry.
- all entries belong to the same Project and Repository;
- the Work's `component_id` equals the primary scope component;
- every changed path matches an AllowedPathRule and belongs to a declared root,
  unless the rule explicitly marks a shared root, root configuration, lockfile,
  migration, or generated output as human-approved;
- scope changes after `branch_created` require a new Component Work revision.

## 5. Shared Packages, Contracts, And Migrations

`packages/contracts/**`, API schemas, database migrations, root workspace
configuration, lockfiles, CI, and code generation are shared paths. They are
never incidental edits in a `single` Component Work.

When a Feature Unit needs a shared path, it creates a coordinated Component
Work with that path declared in AllowedPathRule and the affected Components
listed in ComponentWorkScope. The Work's verification must include producer and
consumer checks, contract compatibility, and the full integration path.

No two active Component Works in the same Repository may write the same
migration, lockfile, root workspace config, generated contract artifact, or
shared package root. The planner serializes or makes the work coordinated.

## 6. Verification Rules

`single` Work runs its Component verification profile plus required affected
package checks. `coordinated` Work runs every participating Component profile,
contract/migration compatibility checks, and the Feature Unit's end-to-end or
human-verification evidence before PR creation.

Targeted checks can provide fast feedback. They do not replace required
cross-component checks when a shared contract, API, migration, or public
behavior changed.

## 7. Public Repository Rules

Public visibility does not relax ADO safety rules. ADO never commits or pushes:

- secrets, tokens, private keys, raw environment files, or credential-bearing
  Git URLs;
- raw Agent/Worker logs, unredacted model output, PII, or production data;
- internal ADO database exports, private packet payloads, or provider billing
  information.

Only reviewed product code, intentionally public documentation, and approved
verification summaries may enter the Project repository. Full operational
evidence remains in ADO artifact storage with its classification and retention
rules.

## 8. Planning And Human Approval

Before a Feature Unit becomes active, the plan must state:

1. affected Components and their monorepo roots;
2. whether each Component Work is `single` or `coordinated`;
3. why a coordinated Work is required, when applicable;
4. branch/PR count, shared-path ownership, and concurrency restrictions;
5. component, contract, integration, and human verification evidence.

The Human Owner approves this decomposition. ADO may not silently split a
coordinated Work into partial PRs or broaden it to shared roots after approval.

## 9. Acceptance Criteria

The managed-project monorepo policy is satisfied when:

1. every Project Component maps to an explicit root in one registered public
   Repository;
2. every Component Work has immutable declared scopes and allowed paths;
3. cross-component atomic functionality uses one coordinated branch and PR;
4. verification proves all affected components together before PR creation;
5. overlapping shared paths are serialized or coordinated;
6. public repository output is screened by the same secret, PII, artifact, and
   external-transfer rules as every other ADO output.

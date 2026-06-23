# ADO Bootstrap Protocol And A1 Entry Criteria

This document defines how ADO is built before ADO can safely orchestrate its
own development. It is the implementation baseline for the separate
`ado-platform` repository, not a template for every managed Project.

## 1. Bootstrap Principle

Bootstrap work is human-governed ordinary engineering. Until the controls in
this document are proven, no agent may use ADO to approve, schedule, execute,
review, or merge ADO code autonomously.

The bootstrap sequence builds trust in this order:

```text
B0: repository and human review discipline
-> B1: durable DB state and deterministic checks
-> B2: observable commands and control-room evidence
-> B3: bounded Worker execution and PR-only Git automation
```

At every stage `main` is protected, `integrate` is the PR target, and the
Human Owner performs final merge outside ADO.

## 2. Required Repository Shape

ADO is one pnpm/Turborepo monorepo with this initial structure:

```text
ado/
  apps/
    api/                 # NestJS REST/OpenAPI/SSE API
    worker/              # NestJS standalone Worker
    control/             # Next.js control room
  packages/
    domain/
    application/
    persistence/
    contracts/
    runtime/
    config/
    testkit/
  infra/
    docker/
  docs/
  package.json
  pnpm-workspace.yaml
  turbo.json
  tsconfig.base.json
  compose.yaml
  .env.example
```

The repository has one lockfile. Root tooling owns TypeScript base options,
formatting, linting, package-manager version, and Turbo task definitions.
Apps may declare only their runtime dependencies and must not own lockfiles or
independent lint/type-check standards.

## 3. Local Dependencies And Configuration

v1 local development requires Node.js at the version pinned in the repository,
pnpm at the version pinned in `package.json`, Git, and PostgreSQL. Docker
Compose may provide PostgreSQL (and optional pgvector later), but it is not
required as the production runtime manager.

No Kafka, Redis, BullMQ, GraphQL server, cloud queue, or hosted provider is a
bootstrap prerequisite.

`packages/config` validates environment variables before API or Worker startup.
The committed `.env.example` contains names and non-secret examples only:

```text
DATABASE_URL=
SESSION_SECRET=
ADO_DATA_DIR=
ADO_WORKTREE_ROOT=
ADO_ARTIFACT_ROOT=
API_ORIGIN=
CONTROL_ORIGIN=
```

Provider credentials, GitHub tokens, production endpoints, and local absolute
user paths never appear in an example file, fixture, screenshot, artifact, or
committed config. The API receives only settings required by the API. Runner
and provider settings are Worker-only.

## 4. Required Root Commands

The exact package-manager scripts are implemented in A1. Their stable intent
is fixed here:

| Command | Required result |
|---|---|
| `pnpm dev:api` | starts the Nest API with validated local configuration |
| `pnpm dev:control` | starts the Next.js control room against the local API |
| `pnpm dev:worker` | starts one Nest standalone Worker with a unique worker ID |
| `pnpm lint` | lints every workspace package |
| `pnpm typecheck` | type-checks every workspace package without emitting artifacts |
| `pnpm test` | runs deterministic unit and module tests |
| `pnpm test:integration` | runs PostgreSQL-backed integration tests in an isolated database |
| `pnpm build` | builds API, Worker, Control, and shared packages |
| `pnpm db:migration:generate` | proposes a TypeORM migration for reviewed local changes only |
| `pnpm db:migration:run` | applies reviewed migrations to the selected non-production database |
| `pnpm db:migration:show` | reports applied and pending migrations |
| `pnpm openapi:generate` | produces the versioned API specification artifact |

`turbo.json` encodes task inputs, outputs, dependencies, and cacheability. Any
task that changes a database, creates a worktree, invokes a provider, sends a
network request, creates a PR, or writes durable artifacts is non-cacheable.

## 5. Database Bootstrap Rules

PostgreSQL is initialized by reviewed TypeORM migrations. `synchronize` is
disabled in all environments. A migration can create ordinary columns and
relations, but PostgreSQL-specific indexes, constraints, triggers, extensions,
and queue locking SQL are explicit and reviewed.

Bootstrap provides separate development, test, and production connection
configuration. Integration tests never share the developer database and never
run against production. Production migration execution is a human-run
operational procedure outside ADO Worker Jobs.

The first migration need only create the minimum health/configuration metadata
required by A1. The full Project/Roadmap/Feature Unit/Job model begins in A2;
A1 must not pretend that the orchestration data model already exists.

## 6. CI Baseline

Every bootstrap PR to `integrate` runs, at minimum:

```text
install with the pinned package manager
-> lint
-> typecheck
-> unit/module tests
-> build
-> migration validation against an empty isolated PostgreSQL database
-> OpenAPI generation and contract-drift check
-> Spec Library manifest/lock validation
```

The CI configuration may not apply production migrations, deploy, push a
branch, create a PR, invoke Codex/Claude/local review models, or use provider
secrets. Build and test artifacts are retained only according to the security
policy and may not contain raw environment files.

## 7. Git And Worktree Bootstrap Rules

The Human Owner creates and protects `main`, creates `integrate`, and grants
the automation identity only the permissions needed to push a work branch and
open a PR to `integrate`. The automation identity has no merge permission.

During bootstrap, branches use the canonical ADO branch form when a Feature
Unit exists. Before ADO creates its own records, manually created bootstrap
branches use this temporary, human-approved form:

```text
ado/bootstrap/{topic}
```

Each branch is reviewed through a PR to `integrate`. ADO-managed worktrees are
introduced only after the A2 repository/worktree records and safeguards exist.
The bootstrap process must not create ad hoc worktrees under user project roots.

## 8. Bootstrap Stages

### B0: Human-Governed Repository

Initialize and protect the ADO Spec Library Git repository, approve and publish
its first immutable manifest, then create the separate `ado-platform`
repository, protect branches, establish `integrate`, and commit an
`ado-spec.lock.json` that pins that manifest. Human review is the only approval
mechanism. No Worker, runner, or API command is trusted yet.

### B1: Executable Technical Base

Implement the monorepo, config validation, PostgreSQL connection, migration
tooling, API health/readiness, Worker process startup/shutdown, Control shell,
and CI baseline. Humans still invoke all commands directly.

### B2: Observable Control Slice

Implement the first durable domain slice, command audit trail, REST read and
command boundary, OpenAPI generation, and a control-room view that proves a
command's resulting record. This is not agent automation; it is proof that the
control plane reports durable truth.

### B3: Bounded Automation Slice

After A4 through A6 are accepted, enable a narrow Worker job that produces an
artifact or runs a deterministic verifier. Only after its lease, timeout,
outbox, audit, and human review evidence are proven may Codex/Git/PR handlers
be introduced. Each handler is separately approved and can be disabled.

## 9. A1: NestJS Monorepo Foundation

A1 is the first implementation Feature Unit. Its outcome is an executable,
testable, observable foundation, not a complete orchestration system.

### 9.1 A1 Scope

A1 includes:

- pnpm workspace and Turbo task graph;
- `apps/api` NestJS application with configuration validation and health/
  readiness endpoints;
- `apps/worker` Nest standalone application context with graceful startup and
  shutdown, but no Job lease loop;
- `apps/control` Next.js shell with API health display and loading/error state;
- shared package boundaries defined in `NESTJS_MONOREPO_ARCHITECTURE.md`;
- TypeORM DataSource configuration, a migration command path, and an initial
  migration proving PostgreSQL connectivity;
- `.env.example`, local Compose configuration for PostgreSQL, root scripts,
  lint/type-check/test/build baseline, and CI checks;
- generated OpenAPI for the health surface and a checked contract artifact.

### 9.2 A1 Explicit Exclusions

A1 does not include:

- Project, Roadmap, Feature Unit, Component Work, Job, or state-machine tables;
- queue leasing, outbox publication, external Runner invocation, or Codex;
- SSE events beyond the API structure necessary to add them later;
- authentication beyond a documented development placeholder boundary;
- local review models, Claude import/export, GitHub PR automation, or merge;
- production deployment, production migration, or direct production access.

### 9.3 A1 Entry Criteria

A1 may begin only when:

1. `ADO_MASTER_SPEC.md`, `NESTJS_MONOREPO_ARCHITECTURE.md`,
   `CONTROL_ROOM_API_UI_SPEC.md`, and this protocol are approved as the current
   implementation baseline;
2. the Human Owner has protected the Spec Library and created `main` and
   `integrate` protection/rules in the separate `ado-platform` repository;
3. the bootstrap repository has a chosen local PostgreSQL method and no secret
   material committed;
4. the A1 Feature Unit has explicit acceptance criteria and a PR target of
   `integrate`;
5. `ado-platform` has a committed, approved `ado-spec.lock.json` for the
   baseline Spec Library revision and implementation work is assigned to a
   human-reviewed bootstrap branch.

### 9.4 A1 Acceptance Criteria

A1 is complete only when a clean checkout can demonstrate all of the following:

1. `pnpm install`, `pnpm lint`, `pnpm typecheck`, `pnpm test`, and `pnpm build`
   succeed using documented local prerequisites;
2. a local PostgreSQL instance receives the initial reviewed migration and
   `pnpm db:migration:show` accurately reports it;
3. the API fails before listening when required configuration is invalid and
   reports health/readiness without exposing secrets when valid;
4. the standalone Worker starts, connects to the intended non-production
   database, records no false job execution, and shuts down gracefully;
5. the Control app shows API health, loading, unavailable, and recovered
   states through the API rather than a database connection;
6. OpenAPI is generated and contract drift fails CI;
7. CI proves all required baseline checks and the work is proposed by a PR to
   `integrate`; no ADO actor merges it.

## 10. Exit To A2

Only a Human Owner accepts A1. A2 may create the first orchestration domain
records after the A1 PR is reviewed, merged to `integrate`, and all A1
acceptance evidence is attached to the Feature Unit. A1 failure evidence is
retained as an Artifact/AuditEvent once A2's data model exists; before then it
is retained in the bootstrap PR and CI logs.

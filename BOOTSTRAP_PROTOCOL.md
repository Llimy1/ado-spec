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

ADO is one Django + Next monorepo with separate Python and frontend tooling:

```text
ado/
  apps/
    api/                 # Django + Django Ninja REST/OpenAPI/SSE API
    worker/              # Python Worker entrypoints
    control/             # Next.js control room
  packages/
    contracts/           # OpenAPI artifact and generated TS client
    ui/                  # optional Control Room UI components
    design-tokens/       # optional generated token artifacts
  infra/
    docker/
  docs/
  pyproject.toml
  uv.lock
  package.json
  pnpm-workspace.yaml
  pnpm-lock.yaml
  Makefile
  .env.example
```

Python dependencies are managed with `uv`. Frontend dependencies are managed
with `pnpm`. Root commands are exposed through `Makefile` or `justfile` so a
developer does not need to remember which package manager owns each task.

## 3. Local Dependencies And Configuration

v1 local development requires Python, `uv`, Node.js, `pnpm`, Git, and
PostgreSQL. Docker Compose may provide PostgreSQL (and optional pgvector
later), but it is not required as the production runtime manager.

No Kafka, Redis, BullMQ, GraphQL server, cloud queue, or hosted provider is a
bootstrap prerequisite.

The Django settings layer validates environment variables before API or Worker
startup.
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
| `make dev-api` | starts the Django API with validated local configuration |
| `make dev-control` | starts the Next.js control room against the local API |
| `make dev-worker` | starts one Python Worker with a unique worker ID |
| `make lint` | runs Python and frontend lint checks |
| `make typecheck` | runs Python type checks where enabled and TypeScript typecheck |
| `make test` | runs deterministic Python and frontend tests |
| `make test-integration` | runs PostgreSQL-backed integration tests in an isolated database |
| `make build-control` | builds the Next.js Control Room |
| `make db-migrate` | applies reviewed Django migrations to the selected non-production database |
| `make db-plan` | reports pending migrations/checks without mutating production |
| `make openapi` | produces the versioned API specification artifact |
| `make generate-client` | regenerates the TypeScript client from OpenAPI |

Tasks that change a database, create a worktree, invoke a provider, send a
network request, create a PR, or write durable artifacts are non-cacheable.

## 5. Database Bootstrap Rules

PostgreSQL is initialized by reviewed Django migrations. Django ORM handles
ordinary model relations. PostgreSQL-specific indexes, partial constraints,
triggers, extensions, and queue locking SQL are explicit reviewed migrations,
typically through `RunSQL` or custom migration operations.

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

## 9. A1: Django + Next Monorepo Foundation

A1 is the first implementation Feature Unit. Its outcome is an executable,
testable, observable foundation, not a complete orchestration system.

### 9.1 A1 Scope

A1 includes:

- Python `uv` workspace/config and frontend `pnpm` workspace;
- `apps/api` Django application with Django Ninja, configuration validation,
  and health/readiness endpoints;
- `apps/worker` Python Worker entrypoint or Django management command with
  graceful startup and shutdown, but no Job lease loop;
- `apps/control` Next.js shell with API health display and loading/error state;
- boundaries defined in `DJANGO_NEXT_PLATFORM_ARCHITECTURE.md`;
- Django database settings, migration command path, and an initial migration
  proving PostgreSQL connectivity;
- `.env.example`, local Compose configuration for PostgreSQL, root commands,
  lint/type-check/test/build-control baseline, and CI checks;
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

1. `ADO_MASTER_SPEC.md`, `DJANGO_NEXT_PLATFORM_ARCHITECTURE.md`,
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

1. `uv sync`, frontend dependency install, `make lint`, `make typecheck`,
   `make test`, and `make build-control` succeed using documented local
   prerequisites;
2. a local PostgreSQL instance receives the initial reviewed migration and
   `make db-plan` or equivalent migration status command accurately reports it;
3. the API fails before listening when required configuration is invalid and
   reports health/readiness without exposing secrets when valid;
4. the Python Worker starts, connects to the intended non-production
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

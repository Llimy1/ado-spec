# ADO NestJS Monorepo Architecture

This document is the canonical implementation architecture for ADO v1. It is
implemented in the separate `ado-platform` repository, not in the ADO Spec
Library repository that contains this document.

It supersedes the Django implementation assumptions in older documents.
ADO remains a PostgreSQL-centered control system: changing framework does not
change the DB single source of truth, state-machine authority, artifact
lineage, outbox, Git safety, or human final authority.

## 1. Technology Decisions

| Concern | v1 Decision | Boundary |
|---|---|---|
| Backend | NestJS with TypeScript | Modular HTTP API and standalone Worker. |
| Repository | pnpm workspace + Turborepo | One Git repository, many apps/packages. |
| Database | PostgreSQL | Sole orchestration truth. |
| Persistence | TypeORM + reviewed SQL migrations | Entity mapping plus PostgreSQL-specific invariants. |
| UI | Next.js App Router | Control room only; no direct DB access. |
| API | REST under `/v1` + generated OpenAPI | Commands and reads have distinct contracts. |
| Live updates | SSE | Best-effort notification; DB reads remain authoritative. |
| Worker queue | PostgreSQL Job/JobAttempt/JobOutboxEvent | No Redis/BullMQ/Kafka in v1. |
| Health | Nest Terminus + Worker DB health records | HTTP readiness and control-room health differ. |
| Vector retrieval | pgvector optional later | Never a control-flow dependency. |

Explicit v1 exclusions:

- Kafka as the system event bus;
- GraphQL and GraphQL subscriptions;
- Redis/BullMQ/Celery/RQ as a queue dependency;
- direct UI-to-database access;
- automatic production migration, deployment, or merge.

## 2. Monorepo Topology

```text
ado-platform/
  apps/
    api/                  # Nest HTTP API: REST, OpenAPI, SSE, health
    worker/               # Nest standalone app: queue and external runners
    control/              # Next.js control room
  packages/
    domain/               # Pure types, state vocabulary, errors, no Nest/TypeORM
    application/          # Use cases, policies, state machine, ports
    persistence/          # TypeORM entities, repositories, migrations, SQL
    contracts/            # API DTOs, OpenAPI types, SSE event payload types
    runtime/              # Runner ports/adapters, process supervisor, Git/Codex
    config/               # Parsed configuration, environment schema, constants
    testkit/              # Factories, fakes, temporary-repository helpers
  infra/
    docker/               # Local Postgres and optional pgvector only
  docs/
  package.json
  pnpm-workspace.yaml
  turbo.json
```

The root workspace owns lockfile, formatting, linting, TypeScript base config,
task graph, and shared development tools. Applications may have local runtime
dependencies but do not create independent lockfiles.

Turborepo coordinates `lint`, `typecheck`, `test`, `build`, and integration
tasks. It may cache deterministic build/test outputs only. It must not cache
database state, artifact/log directories, worktrees, `.env` contents, or any
task that performs a mutable external side effect.

## 3. Application Responsibilities

### 3.1 `@ado/api`

Owns the HTTP boundary only:

- REST `/v1` controllers and DTO validation;
- OpenAPI document generation;
- authenticated SSE streams;
- health/readiness endpoints;
- command-to-application-service mapping;
- response shaping and safe error serialization.

API controllers never mutate TypeORM entities directly, execute Runners, lease
Jobs, or choose a state transition. They invoke application use cases.

### 3.2 `@ado/worker`

Boots a Nest standalone application context, without an HTTP listener. It owns:

- WorkerRegistration lifecycle;
- Job leasing, heartbeat, terminalization, and recovery;
- outbox publishing;
- handler dispatch;
- process supervision and external Runner invocation;
- bounded operational command-line entry points.

The Worker imports the same application/persistence/runtime modules as the API.
It has no controllers and cannot rely on HTTP guards, interceptors, or pipes.
All authorization and validation it needs is explicit in application services.

### 3.3 `@ado/control`

Owns the Next.js control room:

- operational dashboard and human decision interfaces;
- REST query/command client;
- SSE connection and cache invalidation;
- loading, empty, error, blocked, and stale states;
- role-aware presentation only.

The Control app never imports TypeORM, database credentials, domain write
services, or Runner code. It receives data through API contracts.

## 4. Package Dependency Rules

```text
control -> contracts
api -> contracts -> application -> domain
api -> application -> persistence/runtime ports
worker -> application -> persistence/runtime ports
persistence -> domain
runtime -> domain
testkit -> domain/application/contracts
```

Additional rules:

- `domain` has no Nest, TypeORM, Next, Node child-process, or HTTP imports.
- `application` defines ports/interfaces; it does not import concrete TypeORM
  repositories or Codex/GitHub/Ollama clients.
- `persistence` implements data ports and is the only package allowed to own
  TypeORM entities, DataSource, migrations, and PostgreSQL SQL.
- `runtime` implements process/provider ports and receives sanitized requests.
- `contracts` contains transport-safe DTO/event types, never entities or
  secrets.
- `api` and `worker` compose concrete providers through Nest modules.

Circular package dependencies are rejected by lint/architecture tests.

## 5. Nest Module Layout

Both API and Worker compose domain modules with the same names:

```text
core
projects
planning
components
artifacts
documents
execution
verification
reviews
policy
state
gitops
integrations
audit
control_api
health
```

Each module normally contains:

```text
src/
  application/          # commands, queries, use cases, policies
  domain/               # local pure concepts when not shared package-level
  infrastructure/       # TypeORM or external adapter implementation
  api/                  # controller/DTO only in api app
  worker/               # handler wiring only in worker app
  tests/
```

Do not create an all-purpose `common` module. A shared abstraction belongs in
one of `domain`, `application`, `contracts`, `config`, or `testkit` only when
it has a real reusable contract.

## 6. TypeORM And PostgreSQL Rules

### 6.1 Entities And Repositories

TypeORM entities map tables and relations. They must not run external commands,
call services, publish events, apply state transitions, or hide business logic
in subscribers/listeners.

Application services use repositories/ports. Read models are explicit query
services that return DTOs; they are not exposed entities or ad hoc query
builders in controllers.

### 6.2 Migrations

- `synchronize` is disabled in every environment.
- Every schema change is a reviewed TypeORM migration.
- Entity metadata may create ordinary columns/relations; PostgreSQL-specific
  partial unique indexes, check constraints, append-only triggers,
  `CREATE INDEX CONCURRENTLY`, extensions, and queue SQL are explicit migration
  SQL using `QueryRunner`.
- Production migration execution is a human-owned runbook, never an ADO Worker
  Job.
- Migrations are idempotently detectable and are never generated/applied from
  an agent response without review.

### 6.3 Transactions

Use a single TypeORM `QueryRunner` for every atomic state/lease/outbox
transaction. The transaction-scoped manager is the only manager used inside
that transaction. Always release the QueryRunner.

External processes, HTTP calls, model calls, Git, GitHub, file uploads, and
SSE publishing happen after commit. The database writes a JobOutboxEvent or
AuditEvent first; an adapter may notify clients later.

### 6.4 PostgreSQL Queue Query

Job leasing uses an audited parameterized PostgreSQL query with `FOR UPDATE
SKIP LOCKED` through a QueryRunner. It locks the logical Job, StateSubject, and
one queued JobAttempt in the stable order defined by `DATABASE_CONSTRAINTS.md`.
No TypeORM convenience abstraction may weaken the lock or turn it into an
unscoped table scan.

## 7. API, Worker, And UI Process Boundaries

```text
Browser
  -> HTTPS REST/SSE -> api process -> application services -> PostgreSQL
Worker process
  -> application services -> PostgreSQL
  -> runtime adapters -> Codex/Git/tests/local model/GitHub
```

The API process and Worker use separate environment profiles and process
identities. The API does not receive provider credentials by default. The
Worker does not expose HTTP control endpoints. PostgreSQL is the only required
communication mechanism in v1.

## 8. Configuration And Secrets

`packages/config` parses all configuration at startup and returns typed,
redacted configuration objects. Invalid configuration fails startup before the
API listens or Worker leases work.

Rules:

- `.env.example` contains names and safe placeholders only.
- database, API session, provider, GitHub, and external-transfer secrets are
  separate named references;
- the API receives only its own session/auth/database settings;
- the Worker receives a provider secret only in the one child process that
  needs it;
- configuration values are never written to audit/log artifacts unredacted.

## 9. Authentication And Authorization

v1-alpha is single Human Owner operation:

- Control and API are local/private by default.
- API commands require an authenticated Human Owner session and CSRF protection.
- API SSE uses same-origin cookie authentication; no long-lived bearer token is
  placed in a query string.
- Worker identity is not a browser user and has no public HTTP command route.
- API documentation is enabled in local development and otherwise requires the
  same Human Owner authorization.

Multi-user roles, team workflows, and remote OIDC are deliberately deferred.

## 10. Health And Lifecycle

The API exposes liveness and readiness routes through Nest Terminus. Readiness
checks API configuration and PostgreSQL connectivity; it does not claim that a
Worker is healthy. Worker health comes from WorkerRegistration, active leases,
heartbeats, outbox lag, and AuditEvents in the DB.

API and Worker enable Nest shutdown hooks. API shutdown stops accepting new
requests after readiness becomes unavailable. Worker shutdown follows
`WORKER_EXECUTION_CONTRACT.md`: drain, heartbeat owned work, then terminate
only proven owned process groups when necessary.

## 11. Monorepo Component Work Rules

ADO itself is one Repository with these initial Components:

| Component | Allowed root |
|---|---|
| `api` | `apps/api/**` |
| `worker` | `apps/worker/**` |
| `control` | `apps/control/**` |
| `core` | `packages/**` |
| `infra` | `infra/**`, root tool configuration only when explicit |

Normal Component Work changes exactly one component root. Changes that must
atomically alter contracts, migration, shared package, API, and Worker use one
human-approved `platform` Component Work with an explicit multi-root
`allowed_paths` list. Separate parallel branches may not edit the same
migration, generated OpenAPI artifact, workspace root configuration, or shared
package unless a human resolves the dependency first.

One Component Work still produces one branch and one PR to `integrate`.

## 12. Legacy Mapping

The following legacy terms are replaced everywhere in future implementation:

| Legacy term | Canonical Nest term |
|---|---|
| Django UI/API | Nest API + Next Control app |
| Django management command Worker | Nest standalone Worker CLI |
| Django model | TypeORM entity + migration |
| selector | read/query service |
| `transaction.atomic()` | TypeORM QueryRunner transaction |
| Django Admin | restricted internal inspection only; Control is primary UI |
| Django signal | forbidden for core orchestration; explicit use case/outbox only |

`DJANGO_ARCHITECTURE.md` is retained only as historical context and is not a
canonical implementation input.

## 13. Acceptance Criteria

The NestJS monorepo architecture is complete when:

- no current canonical document requires Django for ADO implementation;
- API, Worker, and Control have explicit process/package boundaries;
- TypeORM migrations preserve every PostgreSQL state/evidence/audit invariant;
- UI cannot access DB or Worker-only credentials;
- Worker can use application services without HTTP-only guards/pipes;
- Component Work rules prevent monorepo shared-path conflicts;
- Kafka, GraphQL, and Redis are absent from v1 runtime dependencies while the
  outbox remains a future integration seam.

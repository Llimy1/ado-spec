# ADO Coding Standards

This document defines coding standards for implementing Agent Development Orchestrator (ADO).

ADO code must be boring, explicit, auditable, and easy to stop safely.

## 1. Design Basis

ADO follows these foundations:

- NestJS module design: clear boundaries, explicit providers, and controllers
  that stay at the transport boundary.
- TypeScript monorepo model: packages have one directional dependency graph and
  apps compose concrete adapters only at their edges.
- TypeORM/PostgreSQL transaction model: business state changes use explicit
  `QueryRunner` transaction boundaries.
- TypeScript testing model: deterministic unit, module, integration, database,
  and critical UI tests prove behavior.
- PostgreSQL model: important invariants must be represented with constraints, indexes, and transactions.
- Harness engineering model: LLMs propose actions, but the harness enforces permissions, schemas, state transitions, evidence checks, and audit records.

## 2. Core Principles

### 2.1 Explicit Over Implicit

Do not hide state changes in signals, model `save()` overrides, middleware side effects, or admin hooks.

Allowed:

- explicit service function
- explicit transaction
- explicit state transition request
- explicit audit event

Forbidden by default:

- business-critical ORM subscribers/listeners
- implicit status updates in model methods
- background jobs that mutate state without a transition request
- broad `except Exception` that marks work as successful

### 2.2 Database-Backed Truth

If a fact controls execution, safety, state, review, verification, cost, or branch behavior, it must be stored in the database.

Markdown, logs, command output, and model responses are artifacts. They may inform decisions, but they do not become truth until imported, validated, and linked to a database record.

### 2.3 Narrow Context

Agent prompts and packets must contain only the context required for the current role and work unit.

Large project documents are referenced by stable IDs and hashes. They are not copied into every prompt unless the role needs them.

### 2.4 State Is Special

State changes are never ordinary field assignments.

All important state changes must use:

```text
TransitionRequest -> PolicyDecision -> StateMachine -> StateTransition + AuditEvent
```

### 2.5 Agents Are Untrusted Contributors

Codex, local models, Claude imports, and any future LLM output are treated as claims.

Claims must pass:

- schema validation
- role validation
- policy validation
- evidence validation
- state-machine validation

## 3. Project Code Style

### 3.1 TypeScript

- Use clear names over abbreviations and strict TypeScript types.
- Prefer simple functions, small modules, and explicit request/result types.
- Define interfaces for service, query, policy, repository, and runner ports.
- Keep comments rare and useful.
- Source code comments for ADO and ADO-managed Projects are written in Korean
  by default.
- Raise domain exceptions for expected failures.
- Never swallow errors that should create `SafetyEvent`, `IncidentReport`, `JobFailure`, or `VerificationRun` records.

### 3.1.1 Korean Comment Style

코드 주석은 한국어로 작성한다. 주석은 길게 설명하는 문서가 아니라, 코드를
읽는 사람이 중요한 경계와 의도를 빠르게 확인할 수 있게 하는 짧은 표지판이다.

좋은 주석:

- 파일, 클래스, 함수, 복잡한 블록의 핵심 책임을 한 줄로 표시한다.
- DB 트랜잭션, 상태 전환, 정책 검사, 보안 경계, 외부 실행 경계처럼 실수하면
  위험한 부분의 의도를 짧게 남긴다.
- 코드만 봐서는 드러나지 않는 "왜 이렇게 해야 하는가"를 설명한다.
- ADO 불변 규칙을 지키는 이유를 확인 가능하게 남긴다.

피해야 하는 주석:

- 변수 대입, 반복문, 반환문처럼 코드가 이미 말하는 내용을 반복하지 않는다.
- 긴 튜토리얼, 회고, 추측, TODO 남발을 주석으로 넣지 않는다.
- 주석으로 정책을 새로 만들지 않는다. 정책은 spec과 DB 상태가 소유한다.
- 오래된 주석을 방치하지 않는다. 코드 의미가 바뀌면 주석도 함께 수정하거나
  제거한다.

예시:

```ts
// 상태 소유권은 StateSubject에만 있다. 도메인 row는 상태를 직접 쓰지 않는다.
```

```ts
// 외부 실행은 커밋 이후에만 시작한다. DB 트랜잭션 안에서 CLI를 실행하지 않는다.
```

```ts
// 제출 토큰은 일회성 자격 증명이다. 원문 대신 해시만 저장한다.
```

```ts
// 원문을 먼저 보존해야 스키마 검증 실패도 감사 가능하다.
```

영어 예외는 허용한다. 외부 API 이름, 프로토콜 이름, 에러 코드, enum 값,
라이브러리 고유 용어처럼 영어가 더 정확한 경우에는 영어를 그대로 쓴다.

### 3.2 NestJS And TypeORM

- Controllers and control-room actions are thin.
- TypeORM entities define data shape and relations; reviewed migrations define
  PostgreSQL constraints and indexes.
- Application use cases perform mutations.
- Query services perform read queries.
- Policies make allow/deny decisions.
- Runners interact with external tools.
- State machine code is the only code that applies state transitions.

### 3.3 PostgreSQL

Use database constraints for invariants that must survive bugs, worker retries, and concurrent jobs.

Required by default:

- UUID primary keys.
- Unique constraints for human-readable keys scoped by project.
- Foreign keys for ownership and lineage.
- Check constraints for bounded enums where practical.
- Indexes for queue leasing, state dashboards, artifact lookup, and audit timelines.

Avoid:

- business-critical facts stored only in JSON fields.
- unindexed dashboard filters.
- long transactions around external processes.

## 4. Naming Rules

### 4.1 Modules And Packages

Use plural domain names for Nest modules and workspace packages unless the
domain is naturally singular.

Examples:

- `projects`
- `planning`
- `components`
- `execution`
- `verification`
- `reviews`
- `policy`
- `state`
- `gitops`
- `artifacts`
- `documents`
- `audit`

### 4.2 Services

Service functions use verb-first names:

- `create_project_from_spec`
- `decompose_roadmap`
- `request_state_transition`
- `lease_next_job`
- `record_verification_result`
- `generate_context_packet`
- `create_pull_request`

### 4.3 Selectors

Selector functions use read-oriented names:

- `get_project_overview`
- `list_ready_component_work`
- `find_open_revision_tasks`
- `load_context_packet_sources`

### 4.4 Policies

Policy functions return structured decisions, not booleans:

- `can_start_component_work`
- `can_export_external_packet`
- `can_create_pr`
- `can_promote_claude_import`

## 5. Error Handling

Expected domain failures should become typed exceptions or structured result objects.

Examples:

- `PolicyDenied`
- `InvalidTransition`
- `MissingEvidence`
- `SchemaValidationFailed`
- `RunnerTimeout`
- `UnsafeCommandDenied`
- `RepositoryStateMismatch`

Unexpected failures must be recorded with enough context to reproduce the failure without exposing secrets.

## 6. Logging And Audit

Logs are operational traces. Audit records are product facts.

Every meaningful automated action must produce an audit event:

- who or what initiated it
- what was requested
- what policy decided
- what evidence was used
- what state changed
- what artifacts were created

Raw logs may be rotated. Audit records must remain queryable.

## 7. Idempotency

Worker actions must be safe to retry.

Every job handler must define:

- idempotency key
- retry policy
- timeout policy
- lock or lease strategy
- duplicate-output handling
- failure record behavior

External side effects require recorded external IDs when available.

Examples:

- PR URL
- branch name
- worktree path
- command run ID
- artifact hash

## 8. Forbidden Patterns

ADO implementation must not use:

- direct status assignment outside the state machine
- automatic merge
- direct push to `main` or `integrate`
- production secret capture
- model signals for core orchestration
- unbounded agent context packets
- model output accepted without schema validation
- long DB transactions around CLI calls
- shell command construction from unvalidated user text
- deleting worktrees or branches without a recorded policy decision

## 9. Completion Standard

A code change is complete only when:

- behavior is implemented in the correct layer
- state changes go through the state machine
- policy checks exist where needed
- database constraints protect critical invariants
- tests cover success and important failure cases
- artifacts and audit records are produced where required
- generated documents remain derived from database state

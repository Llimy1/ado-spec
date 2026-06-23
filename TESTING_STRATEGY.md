# ADO Testing Strategy

This document defines the test strategy for implementing ADO.

## 1. Goal

ADO tests must prove that the harness is more reliable than the agents it controls.

The most important failures are not syntax errors. They are unsafe transitions, missing evidence, stale context, incorrect PR targets, unbounded prompts, and false completion claims.

## 2. Test Pyramid

Recommended v1 layers:

```text
unit tests
service tests
database integration tests
runner contract tests
document generation tests
state-machine tests
policy tests
UI smoke tests
end-to-end workflow tests
```

Use Vitest for TypeScript unit and module tests, `@nestjs/testing` for Nest
module composition tests, PostgreSQL-backed integration tests for persistence
and queue behavior, and Playwright only for critical Control-room user flows.
The chosen test runner is never a substitute for the required evidence layers.

## 3. Required Test Categories

### 3.1 Policy Tests

Must cover:

- allowed command
- denied command
- denied branch target
- denied external transfer
- denied stale artifact
- budget denial
- manual override requirement

### 3.2 State Machine Tests

Must cover:

- every allowed transition
- important denied transitions
- actor role mismatch
- missing policy decision
- missing evidence
- duplicate transition request
- audit event creation

### 3.3 Evidence Gate Tests

Must cover:

- verification passed
- verification failed
- review findings block PR
- stale document rejected
- missing artifact rejected
- human verification checklist required

### 3.4 Service Tests

Must cover:

- project creation
- project constraint generation
- roadmap import
- feature unit approval
- component work creation
- job leasing
- context packet generation
- revision task creation
- PR creation request
- Spec Library revision import, deprecation, and revocation handling
- Project revision upgrade creates a new configuration lineage without changing
  prior packet/run evidence

### 3.4.1 Spec Library Boundary Tests

Must cover:

- manifest and `ado-spec.lock.json` schema validation;
- lock commit/hash mismatch rejection;
- unapproved or revoked SpecLibraryRevision rejection for a new packet/run;
- packet/run/verification/PR revision and manifest-hash lineage;
- Worker preflight denial before a Runner starts when Platform lock and packet
  revision do not agree;
- a historical run remains readable after its revision is deprecated or revoked.

### 3.4.2 Managed Project Monorepo Tests

Must cover:

- one active public Repository per managed Project;
- `single` Component Work has exactly one primary ComponentWorkScope;
- `coordinated` Work requires two or more declared scopes in the same
  Repository and creates one branch/PR only;
- changed paths outside declared roots/shared-path approval block PR creation;
- competing Works cannot concurrently claim the same migration, lockfile, root
  configuration, generated contract, or shared package root;
- coordinated verification fails when any required participating Component or
  contract check is absent.

### 3.5 Runner Contract Tests

Runners must be tested without depending on real external model behavior.

Use fake runners for:

- Codex success output
- Codex schema-invalid output
- Codex timeout
- local reviewer success output
- verifier command failure
- GitHub PR creation failure

Real CLI smoke tests may exist, but they must not be the only coverage.

### 3.5.1 Worker Contract Tests

Must cover:

- one Worker leases one eligible Job and creates one active JobAttempt
- competing Workers cannot lease the same Job
- preflight denial launches no runner
- Runner executes outside the lease/state transaction
- valid negative verification evidence terminalizes handler execution without a blind retry
- timeout terminates only the owned process group and retains logs
- SIGTERM drains before lease expiry/recovery
- stale attempt recovery preserves old attempt and reconciles side effects
- outbox transition/job publication is idempotent across a crash boundary
- duplicate Worker key fails closed

### 3.6 Repository Safety Tests

Must cover:

- branch name generation
- `main` rejected as target
- `integrate` direct push rejected
- worktree path outside allowed root rejected
- dirty worktree detection
- PR created only after evidence gates pass

### 3.7 Document Tests

Must cover:

- YAML frontmatter generated
- source version included
- context hash included
- stale document detected
- manual edit detected
- invalid document rejected as agent input
- project constraints included by scope
- unrelated project constraints omitted from context packet

### 3.8 UI Smoke Tests

Must cover:

- dashboard loads
- project detail loads
- feature unit approval screen loads
- component work detail loads
- job log screen loads
- PR status screen loads
- human verification screen loads

UI smoke tests prove the control room can operate. They do not replace service tests.

## 4. End-To-End Workflow Tests

ADO should include one fake end-to-end workflow:

```text
create project
-> approve project constraints
-> import roadmap
-> decompose feature unit
-> approve feature unit
-> create component work
-> fake implementation runner
-> fake verification runner
-> fake local reviews
-> fake arbiter
-> create fake PR record
-> human verification pending
```

This test should use fake integrations and temporary repositories.

## 5. Database Testing Rules

Use database tests for:

- constraints
- unique keys
- foreign keys
- transaction rollback
- queue leasing concurrency
- state transition atomicity
- append-only trigger/permission protection
- stale/quarantined artifact rejection
- project-boundary rejection
- outbox commit and publish recovery

Do not mock the database for code whose purpose is to protect database truth.

The authoritative database test contract is in `DATABASE_CONSTRAINTS.md` and
the lifecycle cases are in `DATA_LIFECYCLE.md`.

## 6. Fixture Rules

Fixtures must be small and explicit.

Recommended factory concepts:

- project
- repository
- component
- roadmap
- feature unit
- component work
- artifact
- context packet
- job
- policy decision
- verification run
- review finding
- pull request

Avoid giant "complete world" fixtures. They hide dependency mistakes.

## 7. External Integration Rules

Default tests must not call:

- real Codex
- real Ollama model
- real GitHub
- real production repositories
- real external notification services

External smoke tests must be opt-in and marked separately.

## 8. Performance And Concurrency Tests

ADO should test:

- job lease contention
- stale heartbeat recovery
- duplicate worker prevention
- dashboard query count for large projects
- artifact lookup indexes

These tests may start as targeted integration tests and become benchmarks later.

## 9. Test Data Safety

Tests must not store:

- real API keys
- real OAuth tokens
- real user PII
- production database rows
- private repository content

If a test needs a secret-like value, use an obvious fake value.

## 10. Definition Of Done For ADO Code

A change to ADO core orchestration is done only when:

- unit tests cover pure logic
- service tests cover business behavior
- database tests cover critical invariants
- runner contracts cover success and failure
- state and policy tests cover safety
- UI smoke test exists for any new control surface
- documentation templates still generate valid packets

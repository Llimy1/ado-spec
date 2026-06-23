# ADO Worker Execution Contract

This document is the canonical execution contract for the ADO v1 Worker.

It implements the boundaries defined in `DB_MODEL_SPEC.md`,
`DATABASE_CONSTRAINTS.md`, `DATA_LIFECYCLE.md`, `RUNTIME_CONSTRAINTS.md`, and
`POLICY_STATE_CONSTRAINTS.md`. A Worker executes work; it does not decide that
agent claims are true and it does not mutate business state directly.

## 1. Scope And Runtime Shape

v1-alpha runtime:

```text
NestJS standalone application context
-> PostgreSQL Job queue
-> one Worker process / one concurrent Job by default
-> ADO-managed local filesystem and POSIX child processes
```

The process supervisor, not the Worker, restarts a stopped Worker. The Worker
does not daemonize itself, fork itself, run database migrations, or auto-scale.

v1-alpha supports macOS and Linux Worker hosts. Windows process-group behavior
is not part of the execution contract until it has an equivalent tested
termination implementation.

Database queue polling is permitted. It is a local database wait and consumes
no model tokens. This must not be confused with agent-context polling. The
default idle interval is 2 seconds and is configurable; PostgreSQL
`LISTEN/NOTIFY` may be added later as a wake-up optimization, not as a source
of truth.

## 2. Non-Negotiable Rules

1. `Job` is the logical state-machine target; `JobAttempt` is immutable
   execution history except for bounded operational fields during its own run.
2. The Worker calls services. It never calls `QuerySet.update()` to set a
   business status.
3. External commands, model calls, Git, and GitHub run outside database
   transactions.
4. Every completed attempt has terminal facts, artifacts, and audit evidence.
   No silent completion is allowed.
5. Runner output is a claim. The Worker validates it structurally and
   semantically before requesting a transition.
6. A lease is not proof of completion. A successful process is not proof of a
   Component Work transition.
7. An uncertain external side effect is reconciled before it is retried.
8. A stopping Worker stops leasing new work first; it does not abandon owned
   child processes without termination or recovery evidence.

## 3. Worker Identity, Capability, And Startup

### 3.1 Identity

`ADO_WORKER_ID` is a stable, unique Worker key. It is registered in
`WorkerRegistration` with:

- Worker key and display labels;
- ADO build/version and supported job types;
- host/OS metadata without secrets;
- start time, heartbeat, and current lifecycle state;
- maximum concurrency and effective runtime profile.

Starting a second active Worker with the same key fails closed. A stale record
may be taken over only after `offline_after_seconds` and an explicit
`--takeover-stale-worker` option; takeover creates an AuditEvent. A Worker key
is not an authorization grant. Policies still decide each lease.

### 3.2 Startup Preconditions

Before any lease, `ado_worker` must:

1. load and validate typed Worker configuration before any lease;
2. load and validate the committed `ado-spec.lock.json`; refuse a missing,
   schema-invalid, or working-tree-modified lock;
3. verify database connectivity and that required migrations are applied;
4. verify the ADO data root, artifact root, worktree root, and temp root are
   present, owned/permissioned as expected, and not symlink escapes;
5. verify configured binaries exist but do not execute them yet;
6. register Worker identity/capabilities and write startup AuditEvent;
7. run one stale-lease and unpublished-outbox recovery scan;
8. refuse startup if an active incident/pause blocks the whole Worker scope.

The Worker never runs `migrate` automatically. A migration mismatch is a
startup failure, not a side effect it repairs itself.

## 4. Management Command Contract

The main entry point is:

```text
pnpm --filter @ado/worker start -- --worker-id <unique-key>
```

Required behavior and options:

| Option | Contract |
|---|---|
| `--once` | Publish one bounded outbox batch and lease/handle at most one Job; exit without sleeping. Used by tests and supervised single-shot execution. |
| `--worker-id` | Overrides `ADO_WORKER_ID`; still must be unique. |
| `--labels` | Comma-separated capability labels; replaces no policy. |
| `--max-concurrent-jobs` | v1-alpha accepts only `1`; any other value fails closed. |
| `--idle-sleep-seconds` | Queue wait interval, validated `0.2..30`. |
| `--drain-seconds` | Grace period after SIGTERM/SIGINT before owned child termination. |
| `--only-job-type` | Repeated restrictive filter for controlled operations; never expands capabilities. |
| `--takeover-stale-worker` | Explicitly performs allowed stale-worker takeover. |
| `--no-startup-recovery` | Forbidden in normal mode; allowed only in isolated tests. |

The command accepts no option for arbitrary shell execution, raw command text,
arbitrary worktree path, arbitrary artifact path, or state override.

`ado_worker` is a thin command wrapper. The loop, lease, handler dispatch,
recovery, and shutdown behavior live in typed execution services.

## 5. Worker Loop

```text
register worker
-> recover stale leases and publish due outbox events
-> while accepting work:
     heartbeat
     publish bounded due outbox batch
     lease exactly one eligible Job
     if none: sleep bounded interval
     else: handle leased JobAttempt
-> drain owned attempt on shutdown
-> mark Worker offline and emit AuditEvent
```

The loop always processes outbox events before looking for ordinary work. It
uses a bounded batch size of 20 in v1-alpha to avoid starving lease/recovery
work. The loop calls `close_old_connections()` before a long idle wait and at
each attempt boundary.

## 6. Lease Contract

### 6.1 Eligibility

A Job is eligible only if all are true:

- logical Job status is `queued`;
- `scheduled_at <= now`;
- a queued JobAttempt exists and attempts remain;
- worker labels/job-type capability match;
- Job target/project/Worker scope has no blocking PauseRecord or Incident;
- required environment and runtime profile are available;
- policy preflight permits lease;
- job type passes the command's restrictive `--only-job-type` filter.

### 6.2 Atomic Lease

`lease_next_job` runs in one short TypeORM `QueryRunner` transaction:

1. select the deterministic highest-priority eligible Job using
   `SELECT ... FOR UPDATE SKIP LOCKED`;
2. lock its StateSubject and one queued JobAttempt;
3. re-check all eligibility after locks are held;
4. create an idempotent system TransitionRequest for `queued -> leased`;
5. persist its PolicyDecision and `job_leased` EvidenceGateResult;
6. apply the Job StateMachine transition;
7. set attempt worker, lease expiration, heartbeat, and `attempt_status=leased`;
8. write lease AuditEvent and commit.

Priority is ascending integer: `0` is highest. Within a priority, order by
`scheduled_at`, then `created_at`, then UUID. The ordering is intentional so
that competing Workers choose predictably when rows are available.

`SKIP LOCKED` is used only for queue consumption. PostgreSQL documents that it
can yield an inconsistent view and is appropriate for avoiding contention among
queue-like consumers, not general product reads.

### 6.3 Lease Timing

Default intervals from `RUNTIME_RULES.md` apply:

```text
heartbeat every 10s
stale attempt after 60s without valid heartbeat/lease extension
worker offline after 180s without Worker heartbeat
```

Lease extension is bounded: `lease_expires_at` may move only while the attempt
is `leased` or `running`, owned by the same Worker, not cancel-requested, and
before the Job hard deadline. A heartbeat never changes a business state.

## 7. Handler Lifecycle

Every handler executes this exact shape:

```text
leased attempt
-> load narrow DB context
-> handler preflight
-> Job leased -> running through StateMachine
-> create AgentRun/CommandRun if applicable
-> execute Runner outside DB transaction
-> persist raw logs and produced Artifacts
-> validate schema, provenance, redaction, and semantic output
-> record domain facts
-> request permitted target transitions
-> terminalize attempt and logical Job through services
-> publish resulting outbox work after commit
```

### 7.1 Preflight

Preflight is deterministic and has no external side effect. It validates:

- handler type and Worker capability;
- Job/attempt target consistency;
- current status and lease owner;
- ContextPacket/ReviewPacket freshness and classification;
- effective SpecLibraryRevision approval/revocation state, manifest hash, and
  compatibility with the Platform `ado-spec.lock.json`;
- allowed paths, verification profile, repository/worktree, and environment;
- budget/provider/network requirements;
- any job-type-specific required artifacts.

Preflight denial terminalizes the Job as `policy_denied`, `blocked`, or
`human_required`, creates the required PolicyDecision/SafetyEvent/AuditEvent,
and never starts a Runner.

### 7.2 Running

The `leased -> running` transition, attempt `started_at`, and run-record
creation occur in a short transaction. A Runner receives an immutable
`RunnerRequest`, not ORM entity instances. The request contains only allowed
paths, resolved argv, sanitized environment references, input artifact
locators, deadline, and correlation identifiers.

### 7.3 Completion Semantics

There are two distinct outcomes:

| Outcome | Job outcome | Example |
|---|---|---|
| Execution failed | terminal Job failure/timeout/blocked state | CLI failed to start, provider unavailable, timeout, invalid schema. |
| Execution completed with negative domain evidence | Job succeeds as a handler execution; target state follows evidence | Verification command reports test failure, review finds P1 issue. |

This distinction prevents a valid test failure from being retried as though the
Worker itself malfunctioned. Negative domain evidence becomes a VerificationRun
failure, ReviewFinding, RevisionTask, or `needs_revision` transition.

### 7.4 Terminalization

The terminal service is idempotent by `(job_id, attempt_number, outcome)`. It
sets the attempt terminal facts, persists/references all artifacts, requests
the logical Job transition through StateMachine, records an AuditEvent, and
creates a retry attempt only if the retry policy explicitly permits it.

No handler may delete a partial worktree, branch, raw log, or artifact to make
terminalization appear clean.

## 8. Runner Contract

Runners are adapters. They receive `RunnerRequest` and return `RunnerResult`.
They do not call Nest application services, mutate status, enqueue jobs, create PR
records, or decide evidence sufficiency.

`RunnerRequest` includes:

- job/attempt/run IDs and correlation ID;
- resolved safe working directory and allowed path scope;
- argv arrays or model input artifact path;
- allowlisted environment values/references only;
- hard deadline and cancellation probe;
- artifact staging directory and log destinations.

`RunnerResult` includes:

- launch/exit facts, duration, signal, and timeout flag;
- staged stdout/stderr/event/output locators and hashes;
- declared output artifact candidates;
- structured result payload locator where applicable;
- normalized failure classification or `null`;
- runner observations, never an applied state.

All command runners use argv execution with `shell=False`. Long stdout/stderr
is written to staging files, not unbounded in-memory pipes. Output becomes a
restricted Artifact before it is selected for any packet or evidence gate.

## 9. Process Supervision And Timeout

The v1 POSIX supervisor starts child commands in a new session/process group.
It writes stdout/stderr to staged files, checks cancellation/deadline at the
heartbeat interval, and updates attempt heartbeat using a dedicated DB-safe
service call.

Timeout/cancellation sequence:

```text
stop accepting new work
-> mark cancellation/timeout intent
-> SIGTERM owned process group
-> wait configured grace period
-> SIGKILL remaining owned process group
-> collect exit/signal/log artifacts
-> terminalize attempt
```

The supervisor never kills a PID it cannot prove belongs to the active attempt.
If ownership cannot be proven, it records `human_required` and an AuditEvent
instead of guessing. A `SIGKILL`-terminated Worker cannot run cleanup; expiry
recovery handles that case.

## 10. Retry And Recovery

### 10.1 Retry Policy

Retry behavior is job-type and failure-class specific. Default timeout/retry
limits live in `RUNTIME_RULES.md` and are upper bounds, not a reason to retry.

- `process_error`, `external_service_failed`, and infrastructure/auth recovery
  may retry when a policy permits and no side effect is uncertain.
- `timeout` retries only where the job type explicitly permits it.
- `verification_failed`, accepted review findings, and implementation failure
  create revision work; they are not blind retries.
- `schema_validation_failed`, policy denial, safety violation, budget denial,
  and human-required outcomes do not auto-retry.

Retry scheduling uses capped exponential backoff with deterministic jitter
derived from Job UUID: 30s, 60s, then 120s maximum unless a job-type policy is
stricter. The new JobAttempt retains the prior attempt link and reason.

### 10.2 Expired Lease Recovery

`ado_recover_workers` scans stale attempts. For each candidate it:

1. locks Job, StateSubject, and Attempt;
2. confirms lease expiry and missing valid heartbeat;
3. verifies no live process ownership evidence exists;
4. records recovery evidence and terminalizes old attempt;
5. creates a retry attempt only if policy permits;
6. emits AuditEvent and updates WorkerRegistration if needed.

Recovery never marks a stale attempt successful. If a process may still own a
side effect, recovery blocks or requires human action before retry.

### 10.3 External Side Effect Reconciliation

Before retrying Git/GitHub side effects, handlers reconcile first:

- worktree/branch: check recorded path, branch, base, and GitSnapshot;
- PR creation: query exact repository + head + base and bind existing external
  PR ID before attempting another create;
- artifact write: verify hash/locator and reuse existing valid artifact;
- external model call with ambiguous completion: preserve run/log evidence and
  require human decision unless the provider gives a safe idempotency key.

## 11. Outbox Publishing

State transitions create `JobOutboxEvent` in their transaction. `ado_worker`
and `ado_publish_outbox` consume due, unpublished events in bounded batches.

For each event, the publisher:

1. locks the event;
2. validates target still permits the intended next Job;
3. creates/reuses Job by `deduplication_key`;
4. records publish AuditEvent;
5. marks the event published in the same transaction.

An outbox publish failure leaves the event unpublished with failure evidence
and a later retry schedule. It never rolls back an already applied state
transition or creates duplicate logical Jobs.

## 12. Shutdown, Health, And Observability

### 12.1 Shutdown

On SIGTERM/SIGINT the Worker enters `draining`:

1. stop consuming outbox and leasing new Jobs;
2. continue heartbeat for the owned attempt;
3. allow it to finish until `--drain-seconds` or its hard deadline;
4. terminate it through the process supervisor if still active;
5. terminalize/release evidence, mark Worker offline, and exit.

SIGKILL/crash recovery is lease-based, not graceful-shutdown based.

### 12.2 Health Signals

The UI derives Worker health from WorkerRegistration, active lease count,
attempt heartbeats, outbox lag, and recent failure classes. It does not trust a
process list alone.

Minimum signals:

- Worker `started`, `ready`, `draining`, `offline`, and last heartbeat;
- active Job/attempt/run and elapsed time;
- queue depth by job type and oldest due job age;
- unpublished outbox count/age;
- timeout, policy, schema, safety, and recovery counts;
- trace/correlation links to all artifacts and audit events.

`WorkerLog` is an Artifact type with `artifact_type=worker_log`; AuditEvent is
the durable timeline. Raw logs are redacted/retained under `DATA_LIFECYCLE.md`.

## 13. Security Boundaries

- Child environments are allowlisted and secret values are not logged.
- Network remains off unless both Job policy and Runner profile allow it.
- Temp/staging/worktree locations are per attempt and must be under ADO roots.
- The Worker never inherits broad provider, database, cloud, or GitHub secrets
  into unrelated children.
- A runner path, argv, artifact locator, model output, repository content, and
  log message are untrusted input until validated.

## 14. Acceptance Criteria

The Worker contract is implemented only when tests prove:

- two Workers cannot lease the same Job;
- a leased Job has a StateMachine transition, policy/evidence records, audit,
  and one active attempt;
- external work never runs inside a DB transaction;
- shutdown and timeout terminate only owned child process groups;
- invalid/stale/restricted input is denied before runner launch;
- valid negative domain evidence does not become a blind retry;
- expired lease recovery preserves history and does not duplicate side effects;
- outbox publication is idempotent and survives a crash between transition and
  Job creation;
- every attempt is traceable from Worker identity to artifacts and transitions.

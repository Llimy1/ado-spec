# ADO Worker Operations

This document defines how ADO Workers are started, observed, drained,
recovered, and diagnosed. It is a v1 operational contract, not a deployment
automation system.

## 1. Ownership Boundary

ADO owns Job records, attempts, artifacts, process supervision, and audit
evidence. The host supervisor owns process restart and machine availability.

ADO v1 does not install a supervisor, deploy containers, modify OS services,
or execute production operations. A human chooses the host mechanism, such as
launchd, systemd, Docker Compose, or a CI runner, and configures it to execute:

```text
pnpm --filter @ado/worker start -- --worker-id <unique-key>
```

The supervisor must use a unique Worker key, preserve the Worker process exit
code, provide SIGTERM on stop, and avoid starting a second active instance with
the same key.

## 2. Required Operational Commands

| Command | Purpose | Side-effect boundary |
|---|---|---|
| `ado_worker` | Normal Worker loop or one-shot execution. | May lease and handle allowed Jobs. |
| `ado_publish_outbox` | Publish bounded due JobOutboxEvents. | May create/reuse Jobs only by event dedupe key. |
| `ado_recover_workers` | Recover expired leases and stale Worker registrations. | May terminalize/retry only with recovery proof. |
| `ado_worker_status` | Print/read Worker, queue, outbox, and incident health. | Read-only. |
| `ado_reconcile_runtime` | Compare DB references to artifacts/worktrees/PRs. | Diagnostic by default; repairs require explicit policy. |
| `ado_enqueue` | Create a known, validated Job from a service request. | No arbitrary argv/path/state input. |
| `ado_collect_stale_jobs` | Compatibility alias for targeted stale-lease report/recovery. | Requires explicit `--apply` for mutation. |

These are logical Nest Worker CLI operations implemented through application
services and structured stdout/stderr. They do not duplicate Worker business
logic or call external tools directly.

## 3. Runtime Configuration

Configuration values are loaded from approved environment/configuration records.
Secret values are never printed or stored in Worker logs.

| Setting | Default | Validation |
|---|---:|---|
| `ADO_MAX_CONCURRENT_JOBS` | `1` | v1-alpha must equal `1`. |
| `ADO_DEFAULT_JOB_TIMEOUT_SECONDS` | `1200` | positive; Job type may be stricter. |
| heartbeat interval | `10s` | less than stale interval. |
| stale attempt threshold | `60s` | greater than two heartbeat intervals. |
| Worker offline threshold | `180s` | greater than stale threshold. |
| idle queue interval | `2s` | `0.2..30s`. |
| outbox batch size | `20` | positive bounded integer. |
| drain grace | `60s` | positive and capped by hard Job deadline. |

Any configuration drift from the registered Worker profile creates an AuditEvent
on next startup. A changed binary path/version does not mutate existing Job
inputs; later Jobs receive a new runtime profile reference.

## 4. Readiness And Health

### 4.1 Readiness

Worker is `ready` only after startup preconditions pass and it has written a
recent WorkerRegistration heartbeat. A process that exists but has no healthy
registration is not ready.

### 4.2 Control-Room Health States

The Next.js Control Room derives these display states through API reads; it
does not invent a second source
of truth:

| UI state | DB evidence |
|---|---|
| `starting` | registration exists, startup audit not yet ready. |
| `ready` | recent heartbeat, no drain flag, capability/config checks passed. |
| `busy` | ready plus one active leased/running attempt. |
| `draining` | Worker drain record/registration state, no new leases. |
| `stale` | heartbeat older than stale threshold. |
| `offline` | stopped/offline state or heartbeat older than offline threshold. |
| `blocked` | whole Worker scope blocked by incident/pause/config failure. |

### 4.3 Minimum Dashboard Data

The control room shows, per Worker:

- key, labels, runtime version, last heartbeat, and current lifecycle state;
- active Job/Attempt, handler, started/remaining timeout, and trace ID;
- queue depth and oldest due Job by type/priority;
- unpublished outbox age/count;
- recent timeout, schema, policy, safety, and recovery events;
- links to WorkerLog artifacts, AgentRun/CommandRun logs, and AuditEvents.

## 5. Alert And Intervention Rules

v1 records intervention-worthy conditions in the UI and AuditEvent timeline.
Notification transport is configured later; it is not an implicit network call.

| Condition | Required system action | Human action |
|---|---|---|
| Worker stale/offline | show degraded Worker, run recovery scan | inspect host/supervisor; restart only after cause is understood. |
| attempt past hard deadline | process termination sequence, timed-out record | inspect evidence; approve retry if policy blocks it. |
| outbox event older than threshold | show backlog and retry publisher | inspect persistent failure/safety condition. |
| repeated failure of one Job type | pause/incident policy evaluation | diagnose runner/configuration before resume. |
| secret/PII/path violation | quarantine + SafetyEvent, incident if critical | resolve or explicitly close incident. |
| duplicate worker key | startup fail closed, AuditEvent | stop conflicting instance or take over stale worker explicitly. |

No alert response may be an automatic main merge, production deploy, or raw-log
export.

## 6. Graceful Shutdown Runbook

1. Send SIGTERM or SIGINT to the Worker process.
2. Confirm its UI state becomes `draining` and no new lease appears.
3. Observe its current attempt until completion or configured drain deadline.
4. Confirm process supervision records termination evidence if it exceeds the
   deadline.
5. Confirm final Worker heartbeat/offline AuditEvent and no active owned child
   process remains.
6. If graceful shutdown was impossible, use `ado_recover_workers` after the
   lease threshold; do not manually mark a Job successful.

`SIGKILL` is an emergency host action. It leaves recovery to the database lease
and process-ownership checks.

## 7. Recovery Runbook

For a stale Worker or attempt:

1. inspect WorkerRegistration, active JobAttempt, heartbeat, and WorkerLog;
2. inspect recorded PID/process group only through the recovery service;
3. inspect worktree/GitSnapshot/PR or external evidence for possible side
   effects;
4. run `ado_recover_workers` in report mode;
5. use `--apply` only when its policy decision/recovery evidence is visible;
6. allow the service to terminalize the old attempt and schedule a new one if
   permitted; otherwise leave it `blocked` or `human_required`.

Recovery evidence must explain why retry is safe. Absence of a heartbeat alone
is insufficient evidence that an external effect did not occur.

## 8. Log And Artifact Operations

- Worker stdout/stderr is operational host output and is mirrored as a bounded
  `worker_log` Artifact where useful.
- Child process stdout/stderr/event data is captured as restricted Artifacts.
- UI displays sanitized summaries first; raw content requires classification
  permission and is never exported to models by default.
- Rotation/retention follows `DATA_LIFECYCLE.md`; deletion writes a tombstone
  AuditEvent and preserves required hash/metadata.
- Trace IDs join Worker log, JobAttempt, AgentRun, CommandRun, artifacts,
  policy decision, state transition, and PR evidence.

## 9. Capacity And Scaling Boundary

v1-alpha default is one concurrent Job per Worker. More Worker processes may
exist only with distinct Worker keys and the same PostgreSQL lease contract.

Before increasing concurrency, prove:

- queue lease contention tests pass on PostgreSQL;
- artifact/worktree paths are per-attempt and cannot collide;
- local model memory/GPU capacity is bounded by labels and policy;
- runner/provider credentials remain scoped;
- outbox lag and recovery behavior remain observable.

Celery/RQ/Redis, distributed rate limiting, automatic scale-out, and remote
artifact storage are future architecture choices. They must preserve the Job,
attempt, outbox, state-machine, and audit contracts rather than replace them.

## 10. Operational Acceptance Criteria

Operations are complete when a human can, from the control room and documented
commands, determine what a Worker is doing, stop it safely, recover a failed
attempt without guessing, and trace every effect to a Job, attempt, artifact,
and AuditEvent.

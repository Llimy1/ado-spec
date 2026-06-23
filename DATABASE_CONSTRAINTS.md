# ADO Database Constraints And Operations

This document makes `DB_MODEL_SPEC.md` enforceable in TypeORM and PostgreSQL.

## 1. Platform

- PostgreSQL is required where an ADO Worker runs.
- Use TypeORM entities and reviewed migrations. PostgreSQL-specific invariants
  are explicit migration SQL, not inferred ORM metadata.
- Use UUID primary keys and UTC `timestamptz`.
- SQLite is allowed only for narrow unit tests, never queue/state integration.
- `pgvector` is optional and deferred until a measured retrieval need exists.

## 2. Constraint Rule

Use database constraints for local facts and transaction services for cross-row,
cross-project, graph, and evidence rules. Both layers are mandatory.

## 3. TypeORM Mapping Rules

Use named PostgreSQL constraints and indexes in reviewed TypeORM migrations for
every local fact PostgreSQL can enforce:

- unique constraints for immutable keys and idempotency;
- partial unique indexes for one active constraint profile, primary component
  mapping, active worktree/pause, and active lease;
- check constraints for positive timeouts, non-negative byte sizes, no
  self-relations, valid paired process fields, and actor-type/user compatibility.

Constraint names begin with the app/domain prefix, for example
`execution_job_project_idem_uniq`, and remain under PostgreSQL name limits.

Delete behavior follows provenance: use `PROTECT` or `RESTRICT` for project,
decision, evidence, and audit lineage; use `SET_NULL` only for nonessential
historical references; and use `CASCADE` only for unapproved private drafts
without audit/execution references. Never bulk-delete or cascade AuditEvent,
StateTransition, PolicyDecision, EvidenceGateResult, HumanDecision, UsageEvent,
or ExternalTransferEvent.

## 4. Required Constraints By Domain

| Domain | Required constraints |
|---|---|
| Identity/project | Actor `(actor_type, actor_key)` unique; SpecLibraryRevision `(repository_identity, commit_sha)` and `manifest_sha256` unique; Project `project_key` unique; one active public Repository and unique Component keys per Project; human actor iff auth user exists. |
| Configuration | one primary ComponentRepository per Component; one active ProjectConstraintProfile per Project; unique profile versions. |
| Planning | Roadmap key unique per Project; FeatureUnit key/sequence unique per Roadmap; acceptance criterion/checklist keys unique per FeatureUnit; dependency rows unique and not self-referential. |
| Component work | ComponentWork key unique per FeatureUnit; ComponentWorkScope component/root unique within work; one primary scope; work relations unique/not self-referential; allowed path rules unique within work. |
| Artifacts | Artifact key unique per Project; bytes non-negative; provenance edges unique/not self-referential; external-safe packet requires passed redaction. |
| Execution | Job key and idempotency key unique per Project; positive attempts/timeout; JobAttempt number unique per Job; one active leased/running attempt per Job. |
| Verification/review | profile command keys unique; one ReviewResult per reviewer/group; one active ArbiterDecision per group; revision source ref required. |
| State/audit | StateSubject unique by Project/type/key; TransitionRequest idempotent; only one StateTransition per request; Audit and external transfer keys unique per Project. |

Every service accepting more than one project-scoped record asserts that all
`project_id` values agree before mutation.

Every packet/run/verification/PR bind operation also asserts that its effective
SpecLibraryRevision is approved, its recorded manifest hash equals that
revision, and it equals the Project revision or an explicitly approved Project
configuration successor. A revoked revision may remain referenced by history
but cannot bind a new executable packet.

For a ComponentWorkScope bind, the service asserts every scope belongs to the
same Project and Repository as the parent Work, matches an approved component
root, and satisfies the `single` or `coordinated` cardinality rule. Shared-path
locks prevent concurrent active Works from changing the same migration,
lockfile, root configuration, generated contract, or shared package root.

## 5. Required Operational Indexes

| Query | Required index |
|---|---|
| Dashboard state list | `StateSubject(project_id, current_status, status_updated_at)`. |
| Feature progress | `FeatureUnit(roadmap_id, sequence_number)` and `ComponentWork(feature_unit_id, sequence_number)`. |
| Queue lease candidate | Job StateSubject `(subject_type, current_status, status_updated_at)` plus Job schedule/priority and queued JobAttempt creation time. |
| Expired lease scan | `JobAttempt(attempt_status, lease_expires_at)`. |
| Run timeline | `(job_attempt_id, started_at)` on AgentRun and CommandRun. |
| Artifact access/retention | `(project_id, artifact_key)`, `content_sha256`, and `(status, retention_until)`. |
| Stale documents | partial DocumentArtifact `(generated_at)` where `is_stale`. |
| Review/revision inbox | ReviewFinding `(status, severity, created_at)` and RevisionTask `(component_work_id, current_status)`. |
| PR dashboard | PullRequest `(component_work_id)` and `(repository_id, base_branch, created_at)`. |
| Audit timeline | AuditEvent `(project_id, occurred_at DESC)`, `(trace_id, occurred_at)`, `(correlation_id, occurred_at)`. |
| Safety inbox | SafetyEvent `(project_id, is_blocking, occurred_at)` and IncidentReport `(project_id, current_status, opened_at)`. |

Use `EXPLAIN (ANALYZE, BUFFERS)` with representative data before adding further
indexes. Every index adds write and storage cost.

## 6. Queue Leasing And Concurrency

`lease_next_attempt(worker)` runs inside one TypeORM `QueryRunner` transaction:

```sql
SELECT job.id
FROM execution_job AS job
JOIN state_state_subject AS subject ON subject.id = job.state_subject_id
WHERE subject.current_status = 'queued'
  AND job.scheduled_at <= now()
ORDER BY job.priority, job.created_at
FOR UPDATE OF job, subject SKIP LOCKED
LIMIT 1;
```

The same short transaction checks worker capability, project/target pause,
incident, policy eligibility, and attempt limits; it then locks the one queued
JobAttempt for the candidate Job, writes Job/attempt lease facts and a lease
AuditEvent, and commits. Codex, tests, Git, GitHub, and local models run only
after commit, outside the database transaction.

`FOR UPDATE SKIP LOCKED` must execute through the transaction-scoped
`QueryRunner` manager. Queue integration tests use separate PostgreSQL
connections/transactions, because test wrappers that share one transaction can
hide locking mistakes. When several rows must be locked, lock in stable
order: Project, StateSubject, Job, JobAttempt, ComponentWork/FeatureUnit, then
UUID lexical order for peers. Retry a PostgreSQL deadlock only at the database
transaction boundary with bounded backoff.

## 7. State And Outbox Transaction

The StateMachine locks StateSubject and TransitionRequest, proves that the
request has not already applied, validates policy/evidence, updates status,
creates StateTransition + AuditEvent + next-job outbox record, and commits.
The worker consumes the outbox after commit. This prevents a claimed state
change from losing the next scheduled action on process crash.

The request idempotency key returns the existing StateTransition when the same
payload was already applied. Reusing the key for different content creates a
SafetyEvent and is rejected.

## 8. Append-Only Protection

1. API/application authorization denies ordinary update/delete of
   append-only tables.
2. PostgreSQL runtime roles deny `UPDATE` and `DELETE` where deployment policy
   permits.
3. A migration-managed trigger rejects update/delete of AuditEvent,
   StateTransition, TransitionRequest, PolicyDecision, EvidenceGateResult,
   HumanDecision, UsageEvent, and ExternalTransferEvent except a privileged,
   audited tombstone/retention operation.
4. Corrections create successor/revocation records rather than editing facts.

Test DBs install the trigger or an equivalent guard. Otherwise append-only is
not proven.

## 9. JSONB And pgvector

JSONB is only for schema-versioned output payloads, policy-rule manifests,
packet selection manifests, provider metadata, and expected-exit-code lists.
Project scope, status, IDs, timestamps, severity, idempotency, and safety
flags must be real columns.

When pgvector is eventually enabled, embeddings belong only to a validated,
non-quarantined Artifact and include model + content hash. Retrieval results
are candidates; they never bypass packet selection, redaction, or evidence
gates. Add an HNSW/IVFFlat index only after recall/latency requirements are
measured.

## 10. Migrations And Database Tests

- Migrations are reviewed Component Work and never auto-run in production.
- Add nullable/table shape first, backfill in bounded idempotent batches, then
  add non-null/check constraints after validation.
- Large indexes use a planned concurrent PostgreSQL migration where supported.
- Backfills and retention actions create AuditEvents.

PostgreSQL integration tests must prove constraints/FKs/partial constraints,
competing-worker lease uniqueness, expired-lease new attempts, atomic
state-transition rollback, append-only rejection, artifact quarantine/external
transfer blocking, cross-project rejection, and rejection of a new executable
packet when its SpecLibraryRevision is revoked or its manifest hash mismatches.

## 11. Acceptance Criteria

This layer is complete only when PostgreSQL integration tests, not merely ORM
metadata validation, prove the constraints, locks, indexes, and
append-only protections described here.

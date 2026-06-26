# ADO Data Lifecycle

This document defines how ADO creates, validates, supersedes, retains,
quarantines, recovers, and deletes database-backed records and artifacts.

The database is the source of truth. Artifact storage and worktrees are
recoverable execution resources addressed by database records.

## 1. Lifecycle Principles

1. Create a new fact instead of mutating history.
2. A record becomes usable only after validation and the required policy gate.
3. Supersession preserves the predecessor and records the relationship.
4. Quarantine removes eligibility for context and evidence, not the audit trail.
5. Retention removes payload access before it removes essential audit metadata.
6. Recovery begins from database records, then verifies external resources.

## 2. Record Classes

| Class | Examples | Mutation Rule | Retention Rule |
|---|---|---|---|
| Configuration | Project, VerificationProfile, ConstraintProfile | Versioned; approval creates active version | Keep active and historic approved versions. |
| Planning | Roadmap, FeatureUnit, ComponentWork | Revision/supersession after execution starts | Keep for project lifetime unless legal policy says otherwise. |
| Execution | Job, JobAttempt, AgentRun, CommandRun | Append-only attempt/run history | Keep summary and evidence metadata; payload by retention class. |
| Artifact | packets, logs, diffs, structured output | Immutable payload, status changes only through service | Expire or delete payload by classification. |
| Decision | policy, evidence, human, arbiter | Append-only; replacement links to prior | Keep for audit lifetime. |
| Audit/Safety | transitions, events, incidents, transfers | Append-only | Longest retention; no routine delete. |

## 3. Artifact Lifecycle

```text
created
-> pending_validation
-> valid
-> superseded | rejected | quarantined | expired | deleted
```

### 3.1 Creation

1. The service creates the database Artifact row with classification,
   `redaction_status`, hash placeholder, and `pending_validation` status.
2. The runner writes payload to an ADO-controlled artifact location.
3. The artifact service computes byte size and SHA-256, stores a non-secret
   locator, and checks type, schema, and redaction requirements.
4. A successful validation creates an AuditEvent and changes the Artifact to
   `valid` through the approved service.
5. A failed validation marks it `rejected` or `quarantined` and creates a
   SafetyEvent when sensitive data is suspected.

No caller can create an immediately valid external or agent-output artifact.

### 3.2 Supersession And Staleness

An Artifact is superseded when a newer authoritative source, context profile,
Git snapshot, or document revision makes it unfit for current use.

- The successor has `supersedes_id` pointing to the older artifact.
- The predecessor becomes `superseded`; its original hash remains immutable.
- Documents and packets calculate `context_hash` from their selected source
  records, constraint profile, and template version.
- If any selected source or profile changes, a stale detector marks the derived
  document/packet `is_stale=true` and it cannot become evidence or new agent
  input.
- Existing JobAttempts retain their original packet for reproducibility. A
  retry with new input is a new Job and new packet, not a mutation.

### 3.3 Quarantine

Quarantine is immediate when secret/PII/production-data detection, a corrupted
payload, policy violation, or provenance failure is suspected.

The quarantine transaction:

1. sets artifact status/classification to quarantined;
2. removes it from packet selection and evidence eligibility;
3. creates a SafetyEvent and AuditEvent;
4. opens or links an IncidentReport when severity is critical;
5. pauses affected scope if policy requires it.

The original bytes are access-restricted. No agent, reviewer, or external
transfer reads a quarantined artifact.

### 3.4 Redaction And External Transfer

Redaction generates a new Artifact. It never overwrites raw content.

```text
restricted source -> redaction artifact -> packet -> human preview -> transfer
```

The redaction artifact has an `ArtifactSourceRef` of `redacted_from`. It is
eligible for a ContextPacket only after redaction validation passes.
An `ExternalTransferEvent` records policy result, destination class, packet
hash, and human preview decision where required. Destination credentials,
tokens, or raw response headers are not stored.

## 4. Planning And Configuration Lifecycle

### 4.1 Project Constraint Profile

```text
draft -> ready_for_human_review -> approved -> active -> superseded | archived
draft -> ready_for_human_review -> rejected
```

Only a human-approved profile can become active. Existing approved Feature
Units and Component Works retain their bound profile version. Rebinding an
active work item requires a HumanDecision and creates an audit event because it
changes execution context.

`rejected` is a terminal draft-review outcome. A rejected profile version is
not reused for later approval; a corrected profile creates a new version or a
new draft record according to the Project configuration service.

### 4.2 Roadmap To Component Work

```text
RoadmapSource valid
-> Roadmap analyzed
-> FeatureUnit/criteria/checklists/component work drafts
-> human approval
-> executable Component Work
```

Roadmap imports are immutable sources. A revised roadmap creates a new source
version and planning review. It does not silently change approved work.

### 4.3 Component Work Revision

Before `branch_created`, a Component Work draft may be edited through an
audited planning service. After that point, changes to repository, branch,
allowed paths, required verification, or constraint profile create a successor
Component Work or an explicit RevisionTask. The historical work remains
traceable and cannot be reinterpreted under a new scope.

## 5. Job And Attempt Lifecycle

`Job` is the immutable logical request; `JobAttempt` is one execution lease.

```text
Job created
-> Attempt queued
-> leased
-> running
-> succeeded | failed | timed_out | cancelled | policy_denied | blocked | human_required
```

### 5.1 Lease

The worker leases one eligible queued attempt in a short database transaction:

1. lock an eligible attempt with `SELECT ... FOR UPDATE SKIP LOCKED`;
2. re-check schedule, pause, incident, worker capability, and max attempts;
3. set worker, lease expiration, heartbeat, and leased status;
4. create the lease AuditEvent;
5. commit before starting any external process.

The worker must never hold a database transaction while running Codex, a test,
Git, GitHub, or a local model.

### 5.2 Heartbeat And Expiry

While an attempt runs, the worker updates only its own heartbeat and lease
expiration at a bounded interval. A recovery worker may reclaim an expired
lease only after it verifies the attempt is not alive or has exceeded the
grace policy. Reclaiming ends the old attempt as `timed_out` or `failed` with
recovery evidence, then creates the next attempt number.

### 5.3 Retry

Retries are policy-driven:

- retryable failure -> new queued `JobAttempt` with incremented attempt number;
- non-retryable failure -> logical Job terminal status projection;
- stale packet/profile/snapshot -> new Job, because inputs changed;
- policy denial, safety violation, or human-required result -> no blind retry.

Idempotency is at Job creation. A duplicate request returns the existing Job;
it never creates a second concurrent logical request.

### 5.4 Process Records

Every external process creates an AgentRun or CommandRun before launch,
captures stdout/stderr as restricted artifacts, stores terminal exit facts, and
then produces a structured runner result. A process success does not update
FeatureUnit or ComponentWork state by itself.

## 6. State And Evidence Lifecycle

```text
claim/result
-> valid artifacts and DB facts
-> TransitionRequest
-> PolicyDecision
-> EvidenceGateResult
-> StateMachine transaction
-> StateTransition + AuditEvent + next Job intent
```

The StateMachine transaction locks the `StateSubject`, verifies the expected
current status and all references, updates `current_status`, inserts one
StateTransition and one AuditEvent, and commits atomically. An outbox-style
`next Job intent` is persisted in the same transaction; the worker creates or
leases the external work only after commit. This prevents a state transition
from claiming work that was never durably scheduled.

If the transition has already been applied for the same TransitionRequest,
the service returns the existing StateTransition. If an idempotency key is
reused with different content, it creates a SafetyEvent and is rejected.

## 7. Review And Revision Lifecycle

```text
verified snapshot
-> ReviewPacket
-> independent ReviewResults
-> ArbiterDecision
-> RevisionTask(s) or PR eligibility
```

Each finding is immutable evidence from one reviewer. The Arbiter records its
own decision rather than rewriting a reviewer finding. An accepted blocking
finding creates a RevisionTask. A resolved RevisionTask points to the new
verification evidence; it is not closed solely because an agent says it is
fixed.

## 8. Git And Pull Request Lifecycle

```text
ComponentWork ready
-> GitWorktree + initial GitSnapshot
-> implementation snapshots/diff artifacts
-> verification and review
-> PullRequest to integrate
-> human verification
-> human merge outside ADO
```

Worktree cleanup is a tracked lifecycle event. Deleting a worktree does not
delete GitSnapshot, diff artifact, branch name, or PR history. A lost worktree
is recoverable from repository, branch, snapshots, and database records; if
recovery cannot prove the expected branch state, the work becomes
`human_required`.

ADO v1 records imported PR merge state when available, but it does not execute
a merge and never targets `main`.

## 9. Retention And Deletion

Retention is configured by Project policy and data classification. Default
classes are:

| Class | Payload Treatment | Metadata Treatment |
|---|---|---|
| public/internal generated docs | Retain while current plus configured history | Keep audit metadata. |
| logs and raw model output | Restricted, short configured retention | Keep hash, classification, run links, and terminal summary. |
| external-safe packets | Retain while needed for reproducibility | Keep transfer event and packet hash. |
| quarantined material | Restricted incident retention | Keep minimum incident/audit metadata. |
| secrets/production data | Reject from storage | Keep only non-sensitive SafetyEvent metadata. |

Payload deletion changes the Artifact to `deleted` or `expired`, clears the
storage locator through a privileged retention service, and records an
AuditEvent. It never silently deletes an Artifact row referenced by a decision,
transition, incident, or audit record. Legal/privacy deletion requirements are
handled by an explicit retention policy and a tombstone record, not a cascade.

## 10. Recovery And Reconciliation

### 10.1 Startup Recovery

On worker startup:

1. register/heartbeat the worker;
2. find expired leases and inspect process ownership evidence;
3. reconcile orphaned runs, worktrees, and artifact locators;
4. create recovery AuditEvents;
5. enqueue only policy-allowed retry or diagnostic jobs.

### 10.2 Reconciliation Rules

- A missing artifact payload makes its Artifact invalid for evidence and opens
  a recovery task; do not recreate a hash by guessing.
- A Git branch/worktree mismatch requires a fresh GitSnapshot and may require
  human review.
- An external PR is synchronized into the existing PullRequest record; a
  changed base branch, head branch, or closed state is audited.
- A partially written structured output remains a restricted raw artifact and
  produces `schema_failed`; it is never promoted from text alone.

## 11. Lifecycle Acceptance Criteria

The lifecycle design is complete when implementation tests prove:

- stale, superseded, rejected, and quarantined artifacts cannot enter packets
  or evidence gates;
- Job retries create new attempts and cannot create duplicate active leases;
- external processes run outside database transactions;
- state, transition, audit, and next-job intent commit together or not at all;
- a Component Work can be reconstructed from DB and artifact/Git references;
- retention preserves required audit metadata while removing payload access;
- incident and pause records stop new affected automation.

# ADO Job Handler Catalog

This document defines the ordinary Job types and Worker control-plane operations
that ADO v1 may execute. Every ordinary Job type has a named handler, a narrow
runner/service boundary, required inputs, output evidence, timeout/retry class,
and side-effect reconciliation rule. Unknown Job types fail closed as
`policy_denied`.

## 1. Handler Interface

Each handler implements a typed interface conceptually equivalent to:

```python
class JobHandler(Protocol):
    job_type: JobType

    def preflight(self, context: JobContext) -> PreflightResult: ...
    def execute(self, request: RunnerRequest) -> RunnerResult: ...
    def persist_result(self, context: JobContext, result: RunnerResult) -> PersistedResult: ...
    def propose_transitions(self, result: PersistedResult) -> list[TransitionRequest]: ...
```

The interface is a service boundary, not permission to mutate state. The Worker
uses handler output to call policy, evidence, artifact, and StateMachine
services. Handlers may not call another handler directly; they emit the next
work through the outbox.

## 2. Common Handler Contract

Every handler must declare:

- `job_type`, supported Worker labels, and v1 release gate;
- target type(s), required packet/artifact types, and required DB bindings;
- timeout, retryable failure classes, and maximum automatic attempts;
- runner or pure-service implementation;
- external side effects and reconciliation procedure;
- required AgentRun/CommandRun/VerificationRun records;
- output schema, semantic validator, and artifact classifications;
- proposed transitions and required evidence gate;
- failure mapping, human-required condition, and audit events.

All handlers share the Worker lifecycle in `WORKER_EXECUTION_CONTRACT.md`.

## 3. v1-alpha Foundation Work

`outbox_publish` and `stale_lease_recovery` are Worker control-plane services,
not ordinary Jobs. They do not create a JobAttempt for themselves: they repair
or publish already durable control-plane records. The two remaining entries are
ordinary Job types.

### 3.1 `document_generation`

| Item | Contract |
|---|---|
| Target | Project, Roadmap, FeatureUnit, ComponentWork, or document artifact scope. |
| Input | Approved DB records, template version, relevant constraint profile. |
| Executor | `DocumentGenerator` service/runner, no product repo mutation. |
| Evidence | DocumentArtifact, frontmatter, source version, context hash, generation log. |
| Timeout/retry | 60s, up to 2 only for process/storage failure. |
| Negative result | Invalid template/data -> handler failure or human-required; never publish stale doc. |
| Reconcile | Reuse a valid artifact only when scope/template/source hash match exactly. |

### 3.2 `artifact_validate`

| Item | Contract |
|---|---|
| Target | Artifact. |
| Input | Pending Artifact locator, declared type/classification, source refs. |
| Executor | Artifact service; no model or network runner. |
| Evidence | Hash, size, MIME/schema/redaction validation result, AuditEvent. |
| Timeout/retry | 60s, one transient storage retry. |
| Negative result | `rejected` or `quarantined`; SafetyEvent if sensitive/corrupt. |
| Reconcile | Hash+locator comparison; never trust a claimed hash. |

### 3.3 `outbox_publish`

| Item | Contract |
|---|---|
| Target | JobOutboxEvent. |
| Input | Unpublished event, deduplication key, current target state. |
| Executor | Outbox publisher service; no external runner. |
| Evidence | Job create/reuse record, publish AuditEvent, event publication timestamp. |
| Timeout/retry | 30s, retry until bounded incident policy says human-required. |
| Negative result | Event remains unpublished with failure artifact; no target rollback. |
| Reconcile | Unique Job `(project, idempotency_key)` and outbox deduplication key. |

### 3.4 `stale_lease_recovery`

| Item | Contract |
|---|---|
| Target | Expired JobAttempt/Job. |
| Input | Lease expiry, worker/attempt heartbeat, process ownership evidence. |
| Executor | Recovery service plus optional diagnostic runner. |
| Evidence | Recovery AuditEvent, old-attempt terminal facts, policy decision. |
| Timeout/retry | 60s per candidate, no blind retry of uncertain side effects. |
| Negative result | `blocked` or `human_required` when a child may still exist. |
| Reconcile | Verify recorded PID/process group and external side effect before next attempt. |

## 4. Planning, Implementation, And Verification Handlers

### 4.1 `codex_planning`

| Item | Contract |
|---|---|
| Release gate | v1-alpha planning support; execution requires human approval. |
| Target | Roadmap / FeatureUnit planning scope. |
| Input | Valid RoadmapSource, narrow planning ContextPacket, planner schema. |
| Runner | Codex Planner, read-only, no network. |
| Evidence | AgentRun, raw event/log Artifacts, schema-valid planner output, generated drafts. |
| Timeout/retry | 20m, one infrastructure-only retry. |
| Negative result | `human_required` for ambiguity; no FeatureUnit approval from model output. |
| Reconcile | Same packet hash may reuse valid output; changed source/profile requires new Job. |

### 4.2 `git_prepare`

| Item | Contract |
|---|---|
| Release gate | v1-beta. |
| Target | ComponentWork. |
| Input | approved work, repository mapping, `origin/integrate`, branch rule, allowed worktree root. |
| Runner | GitWorkspaceManager with approved argv only. |
| Evidence | GitWorktree, initial GitSnapshot, branch/base evidence, CommandRun logs. |
| Timeout/retry | 60s, one retry after reconciliation. |
| Negative result | dirty/unprovable worktree -> human-required. |
| Reconcile | Existing worktree/branch must match recorded base and snapshot exactly. |

### 4.3 `codex_implementation`

| Item | Contract |
|---|---|
| Release gate | v1-beta. |
| Target | ComponentWork. |
| Input | valid implementer ContextPacket, current worktree/snapshot, allowed paths, schema. |
| Runner | Codex Implementer, `workspace-write` only in owned worktree, network off. |
| Evidence | AgentRun, structured output, GitDiffArtifact, changed-path check, secret scan. |
| Timeout/retry | 40m; no blind retry, one infrastructure retry only after worktree reconciliation. |
| Negative result | policy/path/secret issue -> SafetyEvent + blocked/human-required; questions -> human-required. |
| Reconcile | Snapshot/diff/current branch are captured before any rerun; ambiguous edits never overwritten. |

### 4.4 `verification`

| Item | Contract |
|---|---|
| Release gate | v1-beta. |
| Target | ComponentWork / VerificationRun. |
| Input | valid VerificationProfile, GitSnapshot, allowed argv arrays, safe environment. |
| Runner | VerificationRunner; no shell, no deploy/production commands. |
| Evidence | VerificationRun, required CommandRuns, stdout/stderr Artifacts, summary artifact. |
| Timeout/retry | 20m; one retry only for classified transient runner failure. |
| Negative result | test/typecheck failure is valid negative domain evidence: VerificationRun failed and revision flow. |
| Reconcile | Rerun requires same profile version + snapshot; otherwise create new verification Job. |

## 5. Review, PR, And Human-Mediated Handlers

### 5.1 `local_review_single_model`

| Item | Contract |
|---|---|
| Release gate | v1-stable. |
| Target | ReviewGroup. |
| Input | valid ReviewPacket only, reviewer profile, result schema. |
| Runner | local model provider, read-only/no network/no repository write. |
| Evidence | AgentRun, ReviewResult artifact, ReviewFinding records. |
| Timeout/retry | 15m, one provider-crash retry. |
| Negative result | missing reviewer is visible and ReviewGroup becomes partial only if policy allows. |
| Reconcile | one ReviewResult per reviewer/group; never overwrite a valid result. |

### 5.2 `arbiter_review`

| Item | Contract |
|---|---|
| Release gate | v1-stable. |
| Target | ReviewGroup / ComponentWork. |
| Input | completed/allowed-partial ReviewGroup, valid findings, ReviewPacket. |
| Runner | Codex Arbiter, read-only/no network. |
| Evidence | AgentRun, ArbiterDecision artifact, accepted/rejected finding links, RevisionTasks. |
| Timeout/retry | 20m, one infrastructure-only retry. |
| Negative result | ambiguity or incomplete evidence -> human-required, never PR-ready. |
| Reconcile | supersede prior ArbiterDecision; preserve all prior findings/decision artifacts. |

### 5.3 `github_pr_create`

| Item | Contract |
|---|---|
| Release gate | v1-beta. |
| Target | ComponentWork / PullRequest. |
| Input | `ready_for_pr` evidence, ADO head branch, base `integrate`, valid PR body packet. |
| Runner | GitHubPRManager with scoped credential reference. |
| Evidence | PullRequest row, external ID/URL, PR body artifact, manager output schema. |
| Timeout/retry | 2m, up to 2 only after head/base reconciliation. |
| Negative result | auth/network error -> classified failure; base/head mismatch -> SafetyEvent. |
| Reconcile | query exact repository + head + base before and after create; bind existing PR if request response was ambiguous. |

### 5.4 `claude_import`

| Item | Contract |
|---|---|
| Release gate | v1-stable, human-mediated only. |
| Target | imported external Artifact. |
| Input | human-provided response, external transfer linkage, import schema. |
| Runner | ClaudeImportHandler, no Claude SDK/API call. |
| Evidence | CandidateArtifact, source linkage, validation/redaction result, HumanDecision if promoted. |
| Timeout/retry | 30s, no automatic retry. |
| Negative result | malformed/sensitive response -> rejected/quarantined. |
| Reconcile | content hash and transfer event prevent duplicate import. |

## 6. Handler Failure Matrix

| Failure class | Worker action | Automatic retry? |
|---|---|---|
| `process_error` | terminalize run/attempt, retain logs, apply policy | only type-policy transient retry |
| `timeout` | terminate owned process group, retain evidence | explicit type-policy only |
| `schema_validation_failed` | preserve safe raw output, AgentRun schema_failed | no |
| `verification_failed` | persist VerificationRun failure, propose revision transition | no |
| `policy_denied` | terminalize without runner | no |
| `safety_violation` | SafetyEvent, quarantine/incident as needed | no |
| `budget_exceeded` | policy/human-required | no |
| `auth_required` | human-required unless non-secret refresh policy exists | no |
| `dependency_missing` | human-required or approved setup work | no blind retry |
| `external_service_failed` | retain evidence | bounded reconciliation retry |
| `human_input_required` | human-required | no |

## 7. Handler Acceptance Criteria

No Job type is enabled until its handler has:

- a catalog entry matching this document;
- a registered typed handler and explicit Worker capability label;
- deterministic preflight and named policy/evidence gates;
- a fake-runner contract test for success, negative domain result, timeout, and
  external-side-effect ambiguity where applicable;
- idempotent artifact/record persistence and reconciliation procedure;
- traceable AuditEvents and no direct business-state mutation.

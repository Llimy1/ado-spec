# ADO State Transition Rules

This document defines the v1 transition rules for ADO entities.

The State Machine is the only component that applies these transitions.

## 1. Common Rules

All transitions require:

- TransitionRequest
- PolicyDecision
- EvidenceGateResult
- StateTransition
- AuditEvent

Exception and terminal states:

```text
blocked
cancelled
incident_hold
closed
archived
```

Pause is a separate control record, not a status.

## 2. Roadmap States

```text
draft
-> analyzed
-> review_ready
-> approved
-> active
-> completed
-> archived
```

Exception transitions:

```text
* -> blocked
* -> cancelled
* -> incident_hold
incident_hold -> previous_state
```

Required gates:

- `draft -> analyzed`: RoadmapSource valid.
- `analyzed -> review_ready`: RoadmapAnalysis valid.
- `review_ready -> approved`: HumanDecision planning_approved.
- `approved -> active`: at least one approved FeatureUnit.
- `active -> completed`: all required FeatureUnits closed or cancelled by human.

## 3. FeatureUnit States

```text
draft
-> ready_for_human_review
-> approved
-> active
-> implementation_done
-> verification_running
-> review_running
-> needs_revision
-> ready_for_pr
-> pr_created
-> human_verification_pending
-> human_verified
-> closed
```

Allowed exception transitions:

```text
* -> blocked
* -> cancelled
* -> incident_hold
ready_for_human_review -> draft
needs_revision -> active
incident_hold -> previous_state
```

Key rules:

- `approved` requires HumanDecision.
- `ready_for_human_review -> draft` requires a HumanDecision
  `feature_unit_changes_requested`, a non-empty reason, and creation of a new
  FeatureUnitSpec revision. The prior review packet remains immutable audit
  evidence and cannot be reused as approval evidence for the new revision.
- `active` requires all blocking dependencies satisfied or explicitly waived.
- `implementation_done` requires all required ComponentWorks implementation_done or later.
- `ready_for_pr` requires all required ComponentWorks ready_for_pr or pr_created.
- `pr_created` requires all required ComponentWorks have PR created.
- `human_verified` requires required HumanVerificationItems passed or explicitly skipped with reason.
- `closed` requires human verification and PR merge/import policy satisfied.

## 4. ComponentWork States

```text
draft
-> ready
-> branch_created
-> implementation_running
-> implementation_done
-> verification_running
-> verification_failed
-> local_review_running
-> local_review_done
-> arbiter_review_running
-> needs_revision
-> ready_for_pr
-> pr_created
-> closed
```

Allowed exception transitions:

```text
* -> blocked
* -> cancelled
* -> incident_hold
verification_failed -> needs_revision
needs_revision -> implementation_running
incident_hold -> previous_state
```

Key rules:

- `ready` requires approved FeatureUnit and ComponentWorkSpec valid.
- `branch_created` requires GitWorktree and initial GitSnapshot.
- `implementation_running` requires leased implementation Job.
- `implementation_done` requires successful Codex Implementer AgentRun and valid GitDiffArtifact.
- `verification_running` requires VerificationProfile.
- `verification_failed` requires failed VerificationRun.
- `local_review_running` requires passed VerificationRun and valid ReviewPacket.
- `local_review_done` requires ReviewGroup completed or partial policy satisfied.
- `arbiter_review_running` requires local_review_done.
- `needs_revision` requires accepted finding, failed verification, or human revision request.
- `ready_for_pr` requires verification passed, review completed, ArbiterDecision permits PR, changed paths allowed.
- `pr_created` requires PullRequest valid and target integrate.
- `closed` requires PR merge imported or human closure.

## 5. Job States

```text
queued
-> leased
-> running
-> succeeded
```

Exception states:

```text
failed
timed_out
cancelled
policy_denied
blocked
human_required
```

Rules:

- `queued -> leased` requires worker capability and no pause/incident hold.
- `leased -> running` requires preflight passed.
- `running -> succeeded` requires runner result accepted.
- `running -> timed_out` requires timeout evidence.
- `failed` or `timed_out -> queued` is allowed only through named
  `retry_queued` transition when retry policy, recovery evidence, and side
  effect reconciliation permit it.
- retry creates a new `JobAttempt`; it never rewrites a terminal attempt.

## 6. AgentRun States

```text
queued
-> running
-> succeeded
```

Exception states:

```text
failed
timed_out
cancelled
schema_failed
policy_denied
human_required
```

Rules:

- `succeeded` requires process success and schema validation.
- `schema_failed` records invalid model output.
- AgentRun success does not imply ComponentWork success.

## 7. CommandRun States

```text
running
-> succeeded
```

Exception states:

```text
failed
timed_out
cancelled
policy_denied
```

Rules:

- every command has argv, cwd, exit code or signal, stdout/stderr artifact refs.
- command success does not imply verification passed unless command is required and classified by VerificationRun.

## 8. VerificationRun States

```text
queued
-> running
-> passed
```

Exception states:

```text
failed
timed_out
skipped
cancelled
```

Rules:

- `passed` requires all required CommandRuns succeeded.
- `failed` may create RevisionTask candidate.
- `skipped` requires human or policy reason.

## 9. ReviewGroup States

```text
queued
-> running
-> completed
```

Exception states:

```text
failed
partial
cancelled
```

Rules:

- normal risk may complete with 2 of 3 reviewers.
- high/security risk requires 3 of 3 reviewers.
- missing reviewer must be visible to Arbiter and PR body.

## 10. Artifact States

```text
draft
-> valid
```

Other states:

```text
stale
superseded
rejected
quarantined
```

Rules:

- only valid artifacts may be evidence.
- quarantined artifacts block downstream use.
- stale artifacts require regeneration or human acceptance.

## 11. PullRequest States

```text
draft
-> open
-> ready_for_review
```

Terminal/imported states:

```text
merged
closed
unknown
```

Rules:

- ADO creates PR to integrate only.
- ADO does not merge.
- merge state is imported from GitHub.

## 12. HumanVerificationItem States

```text
draft
-> final
-> passed
```

Exception states:

```text
failed
skipped
```

Rules:

- AI cannot mark passed.
- skipped requires reason.
- required items must pass or be explicitly skipped by HumanDecision.

## 13. SafetyEvent States

```text
open
-> acknowledged
-> resolved
```

Other:

```text
false_positive
```

Critical SafetyEvent opens IncidentReport.

## 14. IncidentReport States

```text
open
-> investigating
-> contained
-> resolved
```

Other:

```text
dismissed
```

Open incident pauses related automation.

## 15. ExternalTransferEvent States

```text
pending_review
-> allowed
-> transferred
```

Other:

```text
denied
cancelled
```

Rules:

- transfer requires external_safe and redaction passed.
- restricted artifacts cannot transfer.

# ADO Evidence Gates

This document defines required evidence for important ADO transitions.

Evidence Gate verifies whether a transition has enough valid proof.

## 1. Evidence Principles

```text
Agent claims are not evidence by themselves.
Evidence must be independently checkable.
```

Valid evidence can include:

- DB record with valid status
- Artifact with status valid
- GitSnapshot or GitDiffArtifact
- VerificationRun
- CommandRun
- ReviewResult
- ArbiterDecision
- PullRequest
- HumanDecision
- SafetyEvent/IncidentReport
- ExternalTransferEvent

Invalid evidence:

- stale artifact
- quarantined artifact
- superseded artifact
- unvalidated model output
- missing DB reference
- summary text without source record

## 2. Common Gate Checks

Every EvidenceGate checks:

- target matches TransitionRequest
- evidence refs exist
- artifact status is valid
- source_version/context_hash compatible
- actor/runner type is allowed for evidence type
- evidence is fresh enough
- no open blocking SafetyEvent
- no incident_hold
- no pause flag blocking target
- required human decision exists when needed

## 3. Roadmap Gates

### roadmap_analyzed

Required:

- RoadmapSource valid
- Codex Planner AgentRun succeeded
- RoadmapAnalysis valid

### roadmap_approved

Required:

- HumanPlanningReviewPacket valid
- HumanDecision planning_approved
- no unresolved planning questions

## 4. FeatureUnit Gates

### feature_unit_ready_for_human_review

Required:

- FeatureUnitSpec draft/valid
- AcceptanceCriterion records
- HumanVerificationItem draft/final
- ComponentWork drafts mapped

### feature_unit_approved

Required:

- HumanDecision feature_unit_approved
- acceptance criteria final
- required components known
- dependencies recorded

### feature_unit_changes_requested

Required:

- HumanDecision `feature_unit_changes_requested` with non-empty reason
- current FeatureUnitSpec and HumanPlanningReviewPacket references
- creation request for a new FeatureUnitSpec revision

The previous review packet remains audit evidence only. It is not valid
approval evidence for the new revision.

### human_verification_changes_requested

Required:

- HumanDecision `human_verification_changes_requested` with non-empty reason
- failed required HumanVerificationResult or linked revision request
- visible PullRequest evidence for the affected Feature Unit

The failed verification result is immutable evidence and cannot be replaced or
silently cleared by a later result.

### feature_unit_active

Required:

- FeatureUnit approved
- dependencies satisfied or waived by HumanDecision
- no blocking incident/pause

### feature_unit_implementation_done

Required:

- all required ComponentWorks implementation_done or later
- no required ComponentWork blocked

### feature_unit_ready_for_pr

Required:

- all required ComponentWorks ready_for_pr or pr_created
- no accepted P0/P1 unresolved findings
- no blocking SafetyEvent

### feature_unit_pr_created

Required:

- all required ComponentWorks have PullRequest open/ready_for_review
- PR target integrate
- PullRequestArtifact valid

### feature_unit_human_verified

Required:

- all required HumanVerificationItems passed or skipped with reason
- HumanDecision human_verified
- PR evidence visible to human

## 5. ComponentWork Gates

### component_work_ready

Required:

- FeatureUnit approved
- ComponentWorkSpec valid
- allowed_paths non-empty
- VerificationProfile assigned
- repository mapping valid

### component_work_branch_created

Required:

- GitWorktree record
- branch matches ADO branch pattern
- base is origin/integrate
- initial GitSnapshot valid

### component_work_implementation_done

Required:

- Codex Implementer AgentRun succeeded
- output schema valid
- GitDiffArtifact valid
- changed files checked against allowed_paths
- no critical SafetyEvent
- no unresolved implementer questions

### component_work_verification_running

Required:

- implementation_done
- VerificationProfile valid
- Verification Job queued/leased

### component_work_verification_failed

Required:

- VerificationRun failed/timed_out
- CommandRun logs captured
- failure summary artifact

### component_work_local_review_running

Required:

- VerificationRun passed
- ReviewPacket valid
- GitDiffArtifact valid
- changed paths allowed

### component_work_local_review_done

Required:

- ReviewGroup completed or partial policy satisfied
- ReviewResult artifacts valid
- reviewer missing warning recorded when applicable

### component_work_needs_revision

Required one of:

- VerificationRun failed
- ArbiterDecision needs_revision
- accepted P0/P1 finding
- HumanDecision request_revision
- policy/safety condition requiring revision

### component_work_ready_for_pr

Required:

- VerificationRun passed
- ReviewGroup completed or accepted partial
- ArbiterDecision status ready_for_pr
- no accepted P0/P1 unresolved findings
- changed paths allowed
- PullRequestPacket valid
- no stale/quarantined evidence
- PolicyDecision allowed

### component_work_pr_created

Required:

- PullRequest record
- PR URL
- base branch integrate
- head branch ADO branch
- PR body artifact valid
- GitHubPRManager output valid

## 6. Job Gates

### job_leased

Required:

- queued job
- scheduled_at <= now
- worker capability matches
- target not paused
- no incident_hold
- lease acquired atomically

### job_succeeded

Required:

- runner result valid
- required output artifacts persisted
- no blocking policy issue
- no unhandled exception

### job_timed_out

Required:

- timeout exceeded
- process termination attempted
- CommandRun/AgentRun status recorded

## 7. PR Creation Gate

Required:

- ComponentWork ready_for_pr
- branch pattern valid
- base branch integrate
- changed paths allowed
- required verification passed
- review completed
- ArbiterDecision ready_for_pr
- PullRequestPacket valid
- no stale/quarantined artifacts
- GitHub auth available by reference
- no raw token stored

## 8. External Transfer Gate

Required:

- packet external_safe true
- redaction_status passed
- no restricted artifacts
- no secrets/PII/production data
- human preview completed for Claude interactive
- ExternalTransferEvent allowed

## 9. Incident Gate

When incident is open:

- block new automation for affected scope
- allow only diagnostic, audit, recovery, or human decision jobs
- resume requires HumanDecision or Incident resolved

## 10. EvidenceGateResult

EvidenceGateResult records:

- gate name
- target
- status
- checked evidence
- missing evidence
- rejected evidence
- warnings
- decision reason

Status:

```text
passed
failed
human_required
blocked
```

## 11. Acceptance Criteria

Evidence gates are complete when:

- every important transition has required evidence
- every evidence item is DB-addressable
- stale/quarantined evidence is rejected
- missing evidence blocks or requires human decision
- agent output alone cannot satisfy any completion gate
- gate result is stored and auditable

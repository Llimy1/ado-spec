# ADO Policy and State Constraints

This document defines the policy/state constraint layer for Agent Development Orchestrator (ADO).

Policy and state constraints are the fourth control layer. They decide whether agent outputs, runner outputs, command results, and human actions may affect the database state.

## 1. Design Basis

ADO policy/state constraints are based on:

- Codex and Claude outputs being claims, not authority.
- Codex non-interactive runs producing machine-readable output and event logs, but not DB state.
- Harness engineering: task state, observability, failure attribution, verification, permissions, entropy auditing, and intervention recording are harness responsibilities.
- Harness safety auditing: final outputs are insufficient because resource-access, permission, boundary, and information-flow violations may happen mid-trajectory.
- State-machine design: state changes must be explicit, valid, transactional, and auditable.

References:

- OpenAI Codex manual: sandboxing, non-interactive execution, JSONL events, structured output, rules.
- Anthropic Claude Code CLI: structured output and permission/tool surfaces.
- AI Harness Engineering, arXiv:2605.13357.
- Auditing Agent Harness Safety, arXiv:2605.14271.

## 2. Non-Negotiable Principle

```text
Agents may propose.
Policy may allow.
Evidence must prove.
State Machine applies.
Audit records.
```

No agent, runner, UI button, or integration directly mutates business state.

## 3. State Mutation Pipeline

All meaningful state changes use this pipeline:

```text
Actor output
-> TransitionRequest
-> PolicyDecision
-> EvidenceGateResult
-> StateMachine transaction
-> StateTransition
-> AuditEvent
-> JobOutboxEvent
-> post-commit Job publish
```

If any step fails, no target state changes.

## 4. Authority Separation

| Layer | Authority |
|---|---|
| Agent output | claims and recommendations |
| Runner output | execution result and proposed transitions |
| Policy Engine | allow/deny/human_required/incident_hold decision |
| Evidence Gate | verifies required evidence exists and is valid |
| State Machine | applies transition transactionally |
| Human Owner | final approval and merge authority |

## 5. TransitionRequest Rules

TransitionRequest is the only way to ask for a state change.

Required properties:

- target type
- target key/id
- current status expected by requester
- requested transition
- requested next status
- requester actor
- source job/run/artifact refs
- reason
- evidence refs
- risk level
- idempotency key

Rules:

- Requester cannot apply transition.
- Request must include evidence refs, not just summary text.
- Request must be idempotent.
- Request must be rejected if target current status changed unexpectedly.
- Request must be rejected if any referenced artifact is stale, superseded, rejected, or quarantined.

## 6. PolicyDecision Rules

PolicyDecision evaluates whether the requested action is allowed.

Possible decisions:

```text
allowed
denied
human_required
incident_hold
blocked
```

Policy checks:

- role authority
- target state
- project/component policy
- risk level
- required human approvals
- budget
- provider permission
- branch protection
- allowed paths
- allowed commands
- external transfer status
- secret/PII/production classification
- incident/pause state
- dependency/license policy

PolicyDecision must record:

- decision
- reason
- evaluated rules
- missing requirements
- safety events
- required human action, if any
- expiry, if decision is temporary

## 7. Evidence Gate Rules

EvidenceGate verifies that the transition has enough valid proof.

Evidence rules:

- Evidence must be a valid DB record or valid Artifact.
- Agent claims are not evidence unless backed by independent harness evidence.
- Required evidence must match transition scope.
- Evidence must not be stale or quarantined.
- Evidence must be produced by the correct actor/runner.
- Evidence must be recent enough for the target state.
- Evidence must have matching source_version/context_hash when applicable.

EvidenceGate decisions:

```text
passed
failed
human_required
blocked
```

## 8. StateMachine Rules

StateMachine applies transitions only inside a DB transaction.

Transaction must:

1. lock target row
2. verify current state matches expected state
3. verify PolicyDecision is allowed
4. verify EvidenceGateResult passed
5. update target status
6. create StateTransition
7. create AuditEvent
8. create next Job outbox intent, if configured
9. commit atomically

If any step fails, rollback.

## 9. Agent Output Separation

Agent statuses are role-local.

Example:

```text
codex_implementer.status = succeeded
```

This means:

```text
Codex claims implementation run completed.
```

It does not mean:

```text
ComponentWork.status = implementation_done
```

StateMachine may apply `implementation_done` only after:

- AgentRun succeeded
- schema validation passed
- git diff exists or no-diff reason accepted
- changed paths allowed
- no safety violation
- output target matches Job target

## 10. Human Decision Rules

HumanDecision is required for:

- roadmap approval
- Feature Unit approval
- high-risk policy overrides
- external transfer of sensitive-adjacent context
- production operation runbooks
- human verification passed/failed/skipped
- emergency stop resume

HumanDecision must record:

- actor
- decision
- reason
- scope
- evidence reviewed
- timestamp
- expiry, if override

AI cannot create final HumanDecision records.

## 11. Pause and Incident Rules

Pause is not a state. It is a control flag/record.

Pause scopes:

- Project
- Roadmap
- Feature Unit
- Component Work
- Worker

Incident hold is a status or guard condition that blocks automation until resolved.

Critical SafetyEvent opens IncidentReport and pauses related automation.

## 12. Idempotency Rules

Every TransitionRequest has an idempotency key.

If a transition with the same key was already applied:

- do not apply again
- return existing StateTransition
- record duplicate request as AuditEvent if useful

If the same key maps to different transition data:

- reject
- create SafetyEvent

## 13. Failure and Recovery

Failure must be recorded without corrupting state.

Failure outputs may transition Job/AgentRun/CommandRun statuses, but they do not automatically transition FeatureUnit or ComponentWork unless a policy/state rule explicitly allows it.

Examples:

- schema failure -> AgentRun failed, Job failed
- test failure -> VerificationRun failed, ComponentWork verification_failed
- P0/P1 accepted finding -> ComponentWork needs_revision
- allowed_paths violation -> SafetyEvent, ComponentWork blocked or human_required
- secret leak suspected -> IncidentReport, artifact quarantined, automation paused

## 14. Harness Mapping

| Harness Responsibility | Policy/State Constraint |
|---|---|
| Task state | StateMachine, StateTransition |
| Observability | AuditEvent, PolicyDecision, EvidenceGateResult |
| Failure attribution | failure taxonomy and transition reasons |
| Verification | EvidenceGate required evidence |
| Permissions | PolicyEngine checks |
| Information flow | external transfer, role isolation, artifact status |
| Intervention recording | HumanDecision, ManualOverride, IncidentReport |
| Entropy auditing | source_version, context_hash, stale detection |

## 15. Acceptance Criteria

Policy/state constraints are complete when:

- no runner mutates business state directly
- every state change uses TransitionRequest
- every transition creates PolicyDecision
- every transition checks EvidenceGate
- every transition creates StateTransition and AuditEvent
- every transition is idempotent
- invalid current state rejects transition
- stale/quarantined artifacts cannot be evidence
- Agent output alone cannot prove completion
- high-risk actions require HumanDecision
- Incident hold blocks automation

## 16. Boundary Statement

ADO state must reflect verified system evidence, not model confidence.

If evidence is missing, weak, stale, or contradictory, the transition must fail or require human decision.

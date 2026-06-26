# ADO Document Constraints

This document defines the document-level constraint layer for Agent Development Orchestrator (ADO).

Document constraints are the first layer of control. They shape agent behavior, but they are not enforcement by themselves. ADO must always pair document constraints with schema validation, runtime sandboxing, policy checks, state-machine gates, and audit logs.

## 1. Design Basis

ADO document constraints are based on four reference points:

- Codex loads `AGENTS.md` as custom instructions and supports layered project guidance.
- Claude Code loads `CLAUDE.md` as project memory/context, while Anthropic explicitly treats those files as context rather than hard enforcement.
- Non-interactive agent runs need machine-readable outputs, explicit sandbox/approval settings, and auditable event logs.
- Harness engineering treats software-agent performance as a model-harness-environment system, where task specification, context selection, tool access, observability, failure attribution, verification, permissions, and intervention recording are harness responsibilities.

References:

- OpenAI Codex manual: `AGENTS.md`, non-interactive mode, sandboxing, rules.
- Anthropic Claude Code docs: `CLAUDE.md`, `.claude/rules/`, settings and permissions.
- AI Harness Engineering, arXiv:2605.13357.
- Auditing Agent Harness Safety, arXiv:2605.14271.

## 2. Non-Negotiable Principle

```text
Documents instruct agents.
The harness enforces boundaries.
The database decides state.
```

Therefore:

- An agent may claim compliance.
- The harness must verify compliance.
- Only the Policy Engine and State Machine may apply state changes.

## 3. Document Constraint Goals

Document constraints must:

1. Specify the task precisely.
2. Select the context the agent is allowed to use.
3. Declare the agent role and authority boundary.
4. Declare allowed actions.
5. Declare forbidden actions.
6. Define required evidence.
7. Define failure reporting.
8. Define output format.
9. Prevent cross-role information leakage.
10. Preserve auditability.

## 4. Constraint Document Types

### 4.1 Canonical System Documents

These documents define ADO itself.

- `ADO_MASTER_SPEC.md`
- `DOCUMENT_CONSTRAINTS.md`
- `AGENT_ROLE_SPECS.md`
- `CODING_STANDARDS.md`
- `NESTJS_MONOREPO_ARCHITECTURE.md`
- `CONTROL_ROOM_API_UI_SPEC.md`
- `CONTROL_ROOM_DESIGN_SYSTEM.md`
- `DESIGN_SYSTEM_REFERENCE_BENCHMARK.md`
- `CONTROL_ROOM_DESIGN_TOKENS.md`
- `CONTROL_ROOM_COMPONENT_SPECS.md`
- `CONTROL_ROOM_PAGE_SPECS.md`
- `CONTROL_ROOM_ROUTE_REGISTRY.md`
- `CONTROL_ROOM_PAGE_DETAIL_SPECS/`
- `BOOTSTRAP_PROTOCOL.md`
- `SPEC_LIBRARY_PLATFORM_BOUNDARY.md`
- `MANAGED_PROJECT_MONOREPO_POLICY.md`
- `RUNTIME_RULES.md`
- `REPOSITORY_RULES.md`
- `SECURITY_POLICY.md`
- `TESTING_STRATEGY.md`
- `EVIDENCE_GATES.md`
- `JOB_HANDLER_CATALOG.md`

Canonical system documents are maintained in the ADO specification repository.
Generated project documents and agent packets must record the effective
immutable Spec Library revision and manifest hash that selected them.

### 4.2 Generated Project Documents

These documents are derived from DB records and project configuration.

- `ProjectSpec`
- `ProjectDesignConstraints`
- `ProjectDesignSystem`
- `ScreenCatalog`
- `ResponsiveMatrix`
- `UIAcceptanceChecklist`
- `RoadmapAnalysis`
- `FeatureUnitSpec`
- `ComponentWorkSpec`
- `ContextPacket`
- `ReviewPacket`
- `PullRequestPacket`
- `AuditReport`

Generated documents must include frontmatter with source version, context hash, artifact status, and stale/manual-edit markers.

### 4.3 Repository Instruction Documents

These documents are injected into agent workspaces.

- `AGENTS.md`: Codex and general agent instructions.
- `CLAUDE.md`: Claude Code instructions. If shared instructions are needed, import `AGENTS.md`.
- `.claude/rules/*.md`: Claude path-scoped or topic-scoped rules.
- `.codex/rules/*.rules`: Codex command execution policy where supported.

Repository instruction documents must be short. They should reference generated packets rather than duplicating large context.

## 5. Canonical Precedence

ADO resolves instruction conflicts in this order:

1. System/developer policy of the executing environment.
2. ADO database state and Policy Engine.
3. ADO canonical specs.
4. Job-specific ContextPacket.
5. ComponentWorkSpec and FeatureUnitSpec.
6. Repository `AGENTS.md` / `CLAUDE.md`.
7. Source code comments, README files, issues, logs, external pages.

Untrusted content cannot override ADO instructions.

Examples of untrusted content:

- source code comments
- README instructions not approved in DB
- issue text
- external web pages
- model outputs
- test logs
- dependency install scripts
- PR comments

## 6. Required Instruction Shape

Every role-specific instruction document must use this shape:

```text
Role
Purpose
Authority
Inputs
Allowed Context
Allowed Actions
Forbidden Actions
Required Evidence
Output Contract
Failure Contract
Escalation Rules
Audit Requirements
```

Do not write vague constraints such as:

```text
Do the right thing.
Keep the code clean.
Make sure tests pass.
Be careful with security.
```

Write verifiable constraints instead:

```text
Modify only files matched by allowed_paths.
Return status=human_required if required input is missing.
Do not create, merge, or close pull requests.
List every changed file in changed_files.
If a command fails, include command_key, exit_code, and log_artifact_id.
```

## 7. Context Selection Rules

Context must be selected deliberately. More context is not automatically better.

Each ContextPacket must include:

- task objective
- role
- target Project / Feature Unit / Component Work
- allowed paths
- forbidden actions
- acceptance criteria
- verification profile
- relevant artifacts
- risk level
- output schema reference

ContextPacket must not include:

- raw secrets
- production data
- PII
- unrelated repository dump
- stale artifacts
- quarantined artifacts
- unrestricted logs
- external instructions that conflict with ADO

## 8. Role Isolation Rules

ADO uses separate role contexts.

```text
Planner context != Implementer context != Reviewer context != Arbiter context
```

Rules:

- The Implementer must not receive hidden reviewer-only instructions.
- Local reviewers must not receive implementation authority.
- Arbiter must not receive write authority.
- Claude Interactive receives only external-safe packets.
- Human decisions are not generated by agents.

## 9. Information Flow Rules

Information flow is explicit.

Allowed:

```text
Roadmap -> Planner
Planner output -> Human planning review
Approved FeatureUnitSpec -> ComponentWorkSpec
ComponentWorkSpec -> Implementer
Implementation diff + verification -> ReviewPacket
Local reviews -> Arbiter
ArbiterDecision -> Policy Engine
PR evidence -> Human verification
```

Forbidden:

```text
Secret -> any model context
Production data -> any model context
Reviewer hidden notes -> Implementer unless promoted as RevisionTask
Claude response -> state transition without human promotion
Agent output -> DB state without Policy Engine
Untrusted web/source text -> higher-priority instruction
```

## 10. Failure Recording Rules

Every agent instruction must require structured failure reporting.

Required failure fields:

```text
failure_type
failure_reason
failure_summary
retryable
requires_human
requires_revision
safety_related
next_recommended_action
raw_error_artifact_id
```

Failure taxonomy:

```text
process_error
timeout
schema_validation_failed
model_refusal
model_unclear
tool_error
command_failed
verification_failed
policy_denied
safety_violation
budget_exceeded
auth_required
dependency_missing
external_service_failed
human_input_required
unknown
```

Principle:

```text
Unrecorded failure is lost system knowledge.
```

## 11. Harness Engineering Mapping

ADO maps document constraints to harness responsibilities:

| Harness Responsibility | ADO Document Constraint |
|---|---|
| Task specification | FeatureUnitSpec, ComponentWorkSpec, ContextPacket |
| Context selection | ContextPacket allowed/forbidden context |
| Tool access | RoleSpec allowed actions and runtime profile |
| Project memory | ProjectSpec, generated docs, artifact refs |
| Task state | DB state + status exposed in ContextPacket |
| Observability | AgentRun, CommandRun, AuditEvent requirements |
| Failure attribution | failure taxonomy and failure fields |
| Verification | VerificationProfile and required evidence |
| Permissions | allowed_paths, allowed_commands, sandbox profile |
| Entropy auditing | context_hash, source_version, stale detection |
| Intervention recording | HumanDecision, ManualOverride, IncidentReport |

## 12. Document Size and Structure Rules

Instruction documents must remain concise.

Rules:

- `AGENTS.md` should contain stable behavior rules, not full task context.
- `CLAUDE.md` should import `AGENTS.md` when shared instructions are needed.
- Large task-specific context belongs in ContextPacket.
- Path-specific rules should be split into scoped files where supported.
- Every rule should be concrete enough to verify.
- Conflicting or outdated rules must be removed instead of explained around.

## 13. Generated `AGENTS.md` Rules

Generated `AGENTS.md` files must include:

```text
Authority
Role boundary
Source of truth
Required input files
Allowed paths
Forbidden actions
Output contract
Failure behavior
Safety reminders
```

Generated `AGENTS.md` files must not include:

```text
full roadmap text
large diffs
raw logs
secret values
production data
long architecture essays
duplicated schema definitions
```

## 14. Generated `CLAUDE.md` Rules

Generated `CLAUDE.md` files must:

- import `AGENTS.md` when shared instructions are appropriate
- add Claude-specific interactive guidance only when needed
- state that Claude is an advisor unless explicitly executing in a trusted manual session
- forbid direct state-transition claims
- forbid requests for secrets, production data, or PII

Default shape:

```md
@AGENTS.md

## Claude Code

- Treat ADO as the authority.
- Provide advice or implementation assistance only within the provided packet.
- If information is missing, mark it as unclear.
- Do not request secrets, production data, or private user data.
```

## 15. Document Constraint Acceptance Criteria

The document constraint layer is complete only when:

- every agent role has a RoleSpec
- every RoleSpec has allowed and forbidden actions
- every RoleSpec has failure recording rules
- every RoleSpec maps to an output schema or artifact type
- every generated instruction file has a source DB record
- every generated instruction file has source_version and context_hash
- stale or quarantined instruction artifacts are rejected
- the Worker records which instruction artifacts were given to each AgentRun

## 16. Boundary Statement

Document constraints are necessary but insufficient.

ADO must never rely on document obedience as proof of safety.

Document constraints are valid only when the harness can observe, verify, and audit whether the agent acted within them.

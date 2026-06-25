# ADO Master Spec

## 0. Identity

**Name:** Agent Development Orchestrator (ADO)

ADO is a generic development automation and orchestration system for many projects. It can handle UI/UX work, backend work, apps, web, design assets, research, infra, and documentation.

ADO exists to reduce long, ambiguous context by turning broad roadmaps into explicit, small, reviewable units.

The canonical ADO rules live in the separate ADO Spec Library repository. The
NestJS application lives in the separate `ado-platform` repository and may run
only against an approved, immutable Spec Library revision. The boundary and
revision rules are defined in `SPEC_LIBRARY_PLATFORM_BOUNDARY.md`.

## 1. Hierarchy

```text
Project -> Roadmap -> Feature Unit -> Component Work -> Agent Run
```

- `Project`: top-level product or service.
- `Roadmap`: human-provided source plan.
- `Feature Unit`: smallest human-verifiable user/system goal.
- `Component Work`: implementation/PR work in one repository; it is either
  single-component or coordinated across declared component roots.
- `Agent Run`: auditable execution by Codex, local model, Claude import, system runner, or human.

Feature Unit is the functional unit. Component Work is the implementation/PR unit.

## 2. Source of Truth

- DB is the single source of truth.
- Markdown files are generated or imported working artifacts.
- Stale or quarantined artifacts cannot be used as agent input or evidence.
- ADO-managed worktrees are execution spaces, not source of truth.

## 3. Roles

- `Human Owner`: final approver and final merge authority.
- `Nest API + Next Control`: REST/OpenAPI/SSE control plane and approval surface.
- `Worker`: job executor.
- `Policy Engine`: permission and safety gate.
- `State Machine`: only component that applies state transitions.
- `Codex Planner`: roadmap decomposition and spec drafting.
- `Codex Implementer`: scoped implementation in ADO worktree.
- `Codex Review Arbiter`: final review synthesis from local reviews.
- `Local Review Council`: Qwen Coder, Devstral, Llama.
- `Claude Interactive`: human-mediated external advisor.
- `System Verifier`: deterministic command/test runner.
- `GitHub PR Manager`: branch push and PR creation, no merge.

No actor may approve its own work as final.

## 4. Safety Boundaries

- main is protected. ADO never modifies or targets main.
- integrate is PR target only. ADO never pushes directly to integrate.
- Work branches use:

```text
ado/{project_key}/{work_type}/{feature_unit_key}/{component_work_key}
```

- ADO creates PRs only. Humans merge.
- Secrets, PII, production data, raw tokens, and billing credentials are never stored or exported.
- Production deploys, production DB access, destructive infra changes, and automatic merge are out of v1.
- Allowed paths and allowed commands are enforced.
- All external transfers require redaction and ExternalTransferEvent.

## 5. Execution Loop

```text
Roadmap
-> Planning
-> Human planning approval
-> Component Work creation
-> Branch/worktree creation
-> Codex implementation
-> Verification
-> Local Review Council
-> Codex Arbiter
-> Revision loop if needed
-> PR creation to integrate
-> Human verification
-> Human merge outside ADO
```

ADO v1 stops at PR creation and human verification workflow. Merge remains human-controlled.

## 6. State Model

Important state changes go through:

```text
TransitionRequest -> PolicyDecision -> StateMachine -> StateTransition + AuditEvent
```

Feature Unit core states:

```text
draft -> ready_for_human_review -> approved -> active
-> implementation_done -> verification_running -> review_running
-> needs_revision -> ready_for_pr -> pr_created
-> human_verification_pending -> human_verified -> closed
```

Component Work core states:

```text
draft -> ready -> branch_created -> implementation_running
-> implementation_done -> verification_running -> verification_failed
-> local_review_running -> local_review_done -> arbiter_review_running
-> needs_revision -> ready_for_pr -> pr_created -> closed
```

Exceptions include:

```text
blocked | cancelled | incident_hold
```

Pause is a flag/record, not a status.

Policy/state constraints are defined in:

- `POLICY_STATE_CONSTRAINTS.md`
- `STATE_TRANSITION_RULES.md`
- `EVIDENCE_GATES.md`
- `AGENT_INGEST_PROTOCOL.md`

## 7. Core Data Model

Core tables:

- Actor, Project, Component, Repository, ComponentRepository, EnvironmentProfile,
  ProjectConstraintProfile
- Roadmap, RoadmapSource, FeatureUnit, AcceptanceCriterion,
  HumanVerificationItem, HumanVerificationResult, FeatureUnitRelation
- ComponentWork, ComponentContract, ComponentWorkRelation, AllowedPathRule
- Artifact, ArtifactSourceRef, DocumentArtifact, ContextPacket, ReviewPacket,
  ArtifactEmbedding
- Job, JobAttempt, JobOutboxEvent, WorkerRegistration, AgentRun, CommandRun,
  VerificationRun, VerificationProfile, VerificationCommand
- ReviewGroup, ReviewResult, ReviewFinding, ArbiterDecision, RevisionTask
- HumanDecision, ApprovalEvent, ManualOverride
- StateSubject, TransitionRequest, PolicyDecision, EvidenceGateResult,
  StateTransition, PauseRecord, AuditEvent
- BudgetPolicy, UsageEvent, SafetyEvent, IncidentReport
- GitWorktree, GitSnapshot, PullRequest, ExternalTransferEvent

Use UUID primary keys. Human-readable keys are separate.

The canonical implementation contract is defined in:

- `DB_MODEL_SPEC.md`
- `DATABASE_CONSTRAINTS.md`
- `DATA_LIFECYCLE.md`
- `DATABASE_ERD.md`

## 8. Artifacts

Artifact classes:

- Source Record
- Generated Artifact
- Human Artifact
- External Artifact

Important artifacts:

- RoadmapSource, RoadmapAnalysis, FeatureUnitSpec, ComponentWorkSpec
- ContextPacket, ReviewPacket, LocalReviewResult, ArbiterDecision
- ImplementationArtifact, GitDiffArtifact, VerificationRun, CommandRunLog
- PullRequestPacket, PullRequestArtifact, HumanVerificationResult
- OperationRunbook, RollbackPlan, SafetyEvent, IncidentReport, AuditReport

Every artifact has source refs, actor, version/hash, status, and policy/redaction metadata.

Structured output schema rules are defined in `SCHEMA_CONSTRAINTS.md`.

## 9. Documents

Markdown is DB-derived unless explicitly imported.

Document constraints are defined in `DOCUMENT_CONSTRAINTS.md`.

Agent role contracts are defined in `AGENT_ROLE_SPECS.md`.

Generated docs include YAML frontmatter:

```yaml
ado_doc_type:
ado_id:
human_key:
source_version:
context_hash:
status:
stale:
generated_from_db: true
manual_edit_detected:
generated_at:
```

Only `valid` documents may be used as input or evidence.

## 10. Code And Project Constraints

ADO has two different constraint layers:

```text
ADO implementation constraints
Project/service constraints
```

ADO implementation constraints define how this orchestration system is built. They are defined in:

- `CODING_STANDARDS.md`
- `NESTJS_MONOREPO_ARCHITECTURE.md`
- `CONTROL_ROOM_API_UI_SPEC.md`
- `CONTROL_ROOM_DESIGN_SYSTEM.md`
- `BOOTSTRAP_PROTOCOL.md`
- `SPEC_LIBRARY_PLATFORM_BOUNDARY.md`
- `MANAGED_PROJECT_MONOREPO_POLICY.md`
- `SERVICE_LAYER_RULES.md`
- `TESTING_STRATEGY.md`

Project/service constraints define how each target product should be designed, implemented, and verified. They are generated per Project from a human-approved ProjectConstraintProfile.

The ADO Control Room design system and every managed Project design system are
separate products. They do not visually inherit from one another. Only the
shared quality baseline (accessibility, responsive integrity, explicit state,
error/recovery behavior, and evidence-based UI verification) applies to both.

Project constraint generation is defined in:

- `PROJECT_CONSTRAINT_GENERATION.md`
- `PROJECT_DESIGN_GOVERNANCE.md`
- `SERVICE_CONSTRAINT_QUESTIONNAIRE.md`
- `templates/PROJECT_DESIGN_CONSTRAINTS_TEMPLATE.md`
- `templates/PROJECT_ARCHITECTURE_TEMPLATE.md`
- `templates/PROJECT_VERIFICATION_PROFILE_TEMPLATE.md`
- `schemas/project_constraint_profile.schema.json`

Constraint precedence:

```text
ADO common constraints
> Project constraints
> Feature Unit constraints
> Component Work constraints
> Agent suggestions
```

ADO common constraints are stable. Project constraints vary by service.

## 11. Worker

v1-alpha uses:

```text
NestJS standalone Worker + PostgreSQL DB queue
```

v1-stable may add optional wake-up, scale, or remote artifact adapters only if
they preserve the Job, JobAttempt, JobOutboxEvent, StateMachine, and audit
contracts. PostgreSQL remains the required queue source of truth.

Worker responsibilities:

- lease jobs
- heartbeat
- precheck policy/state
- call runners
- persist artifacts
- request transitions
- publish approved JobOutboxEvents
- record audit events

Worker does not directly mutate business states.

The canonical Worker contract is defined in:

- `WORKER_EXECUTION_CONTRACT.md`
- `JOB_HANDLER_CATALOG.md`
- `WORKER_OPERATIONS.md`

Runners:

- CodexRunner
- LocalModelRunner
- VerificationRunner
- GitWorkspaceManager
- GitHubPRManager
- DocumentGenerator
- ClaudeImportHandler

Runtime enforcement rules are defined in:

- `RUNTIME_CONSTRAINTS.md`
- `RUNTIME_RULES.md`
- `REPOSITORY_RULES.md`
- `SECURITY_POLICY.md`

## 12. Codex CLI

ADO uses `codex exec` for non-interactive automation.

Role isolation:

- Planner: read-only
- Implementer: workspace-write
- Arbiter: read-only

Default command pattern:

```bash
codex exec \
  --cd {workdir} \
  --sandbox {read-only|workspace-write} \
  --ask-for-approval never \
  --output-schema {schema_file} \
  --output-last-message {output_file} \
  --json \
  --ephemeral \
  -
```

Danger full access and bypass flags are forbidden by default.

AGENTS.md contains short execution rules only. Long context lives in ContextPacket.

## 13. Claude Interactive

Claude is v1 human-started, not an automatically executed SDK/MCP agent.
Claude Code may submit its result directly to ADO through the scoped Agent
Ingest API. It never receives database credentials and never writes DB rows
directly.

Flow:

```text
ADO exports external-safe packet
-> human pastes into Claude
-> Claude Code posts result to ADO Ingest API
-> ADO stores raw and structured artifacts
-> ADO validates result and creates follow-up Job when policy allows
-> Worker leases the Job and runs Codex
```

Claude responses are candidate artifacts, not state-transition evidence.

The canonical ingest contract is defined in:

- `AGENT_INGEST_PROTOCOL.md`
- `schemas/agent_ingest_result.schema.json`
- `schemas/claude_import_output.schema.json`

Claude SDK and Claude MCP automatic integration are excluded from v1.

## 14. Local Review Council

Reviewers:

- Qwen Coder: code, logic, API, tests.
- Devstral: task completion and multi-file consistency.
- Llama: UX, requirement, human checklist perspective.

Normal risk: 2 of 3 reviewers may complete ReviewGroup.

High/security risk: 3 of 3 required.

Local reviewers find issues. Codex Arbiter decides which findings are accepted, rejected, merged, or escalated.

P0/P1 accepted findings block PR.

## 15. Verification

ADO does not trust implementation claims. It trusts evidence.

VerificationProfile defines allowed commands and human checks.

Commands are stored as argv lists.

Component Work Verification gates PR creation. Feature Unit Verification gates human verification.

Risk levels:

```text
low | normal | high | security_sensitive | production_data_related
```

Testing may be command-based or artifact-review-based.

## 16. GitHub / PR

Branch from integrate. PR to integrate.

PRs are Component Work level. A single Work has one component root; a
coordinated Work has multiple declared roots but still one branch and one PR.
Feature Unit UI groups related PRs.

PR creation requires:

- ComponentWork ready_for_pr
- branch rule valid
- base branch integrate
- changed paths within allowed paths
- required verification passed
- review group completed
- arbiter permits PR
- PullRequestPacket valid
- no stale/quarantined artifacts

ADO does not merge.

## 17. UI

Database inspection tooling is not the control room. The Next.js Control app
uses the Nest REST/OpenAPI/SSE boundary and never writes the DB directly.

Essential screens:

- Project
- Roadmap
- Feature Unit
- Component Work
- Runs / Logs
- Reviews
- PRs
- Artifacts
- Incidents
- Human Decision Inbox
- Settings

UI buttons enqueue jobs or create HumanDecision records. They do not run workers directly.

## 18. Operations

All events are traceable via trace_id/correlation_id.

Logs:

- AuditEvent
- AgentRunLog
- CommandRunLog
- WorkerLog
- PolicyDecisionLog
- StateTransitionLog
- SafetyEventLog
- UsageEventLog
- ExternalTransferLog
- IncidentLog

AuditEvent is append-only.

Incident opens pause related automation.

Unknown costs remain unknown. Do not invent cost numbers.

## 19. v1 Roadmap

### v1-alpha

Goal: DB/state/worker/UI/artifact foundation.

Feature Units:

- FU-A1 NestJS monorepo/PostgreSQL bootstrap
- FU-A2 Core DB models
- FU-A3 State Machine / Policy Engine
- FU-A4 DB Job Queue + Worker
- FU-A5 ArtifactStore + Markdown generation
- FU-A6 Next.js Control Room + Nest API operational slice

### v1-beta

Goal: implementation, verification, and PR loop.

Feature Units:

- FU-B1 Git Workspace Manager
- FU-B2 VerificationRunner
- FU-B3 CodexRunner Implementer
- FU-B4 PR Body + GitHub PR Manager
- FU-B5 End-to-End Beta Loop

### v1-stable

Goal: review quality, revision loop, Claude import, and operations.

Feature Units:

- FU-S1 Local Review Council
- FU-S2 Codex Arbiter
- FU-S3 Revision Loop
- FU-S4 Claude Interactive Import
- FU-S5 Safety / Incident / Budget Monitoring
- FU-S6 Stable UI Polish

## 20. v1 Exclusions

- automatic merge
- production deploy
- production DB access
- Claude SDK automation
- Claude MCP automatic integration
- Gemini/Grok/Copilot paid reviewer APIs
- multi-user team approval workflow
- Celery/RQ/Redis as hard dependency
- Kubernetes operation
- full visual regression platform
- automatic dependency upgrade
- automatic GitHub review comment resolve
- direct database writes from external agents

## 21. Success Definition

ADO v1 is successful when one real project can:

```text
Roadmap -> Feature Unit decomposition -> human approval
-> Component Work branch/worktree
-> Codex implementation
-> verification
-> Local Review Council + Arbiter
-> PR to integrate
-> human verification workflow
```

v1 complete does not mean automatic main merge.

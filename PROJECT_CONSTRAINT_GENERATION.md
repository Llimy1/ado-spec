# Project Constraint Generation

This document defines how ADO creates and manages project-specific constraints.

ADO is generic. Every real product needs its own service identity, design rules, architecture rules, verification profile, and human checklist standards.

## 1. Principle

ADO common constraints define what the orchestration system is allowed to do.

Project constraints define what the target service should become.

They are different layers.

### 1.1 Design-System Separation

ADO has one Control Room design system for its own operating interface. Each
managed Project has an independent Project Design System. A Project does not
inherit the Control Room's colors, typography, layout, components, voice, or
motion. Conversely, Project branding never changes the Control Room.

Both layers must meet the Shared Quality Baseline defined in
`PROJECT_DESIGN_GOVERNANCE.md`: accessibility, responsive integrity, explicit
system state, loading/error/recovery states, and evidence-based UI verification.

## 2. Constraint Layers

Precedence:

```text
ADO common constraints
> Project constraints
> Feature Unit constraints
> Component Work constraints
> Agent suggestions
```

Examples:

- A project may prefer a playful visual tone, but ADO security policy still forbids leaking secrets.
- A Feature Unit may request a new UI pattern, but project design constraints still define the product tone.
- A Component Work may need a backend shortcut, but project architecture constraints still define API and database rules.

## 3. Project Constraint Profile

Each Project has one active `ProjectConstraintProfile`.

Required fields:

- project identity
- target users
- product tone
- design constraints
- architecture constraints
- component map
- verification profile
- safety additions
- forbidden project-specific behavior
- human-approved status
- source answers version
- generated document hashes

Profiles may be versioned. Only one version is active for new work.

## 4. Generation Flow

```text
1. Human creates Project
2. ADO generates Service Constraint Questionnaire
3. Human answers questionnaire
4. Codex Planner drafts ProjectConstraintProfile
5. For UI-bearing components, ADO creates or imports a Project Design System
6. ADO generates project constraint docs and a Design Contract candidate
7. Human reviews the design/architecture/verification baseline
8. Human approves ProjectConstraintProfile and the active Design Contract
9. ADO marks constraints active
10. Future ContextPackets include only relevant constraint excerpts
```

## 5. Required Project Constraint Documents

For each project, ADO generates:

- `PROJECT_SPEC.md`
- `PROJECT_DESIGN_CONSTRAINTS.md`
- `PROJECT_DESIGN_SYSTEM.md` for UI-bearing Projects
- `SCREEN_CATALOG.md` for UI-bearing Components
- `RESPONSIVE_MATRIX.md` for UI-bearing Components
- `UI_ACCEPTANCE_CHECKLIST.md` for UI-bearing Components

The generated document structure comes from:

- `templates/PROJECT_DESIGN_CONSTRAINTS_TEMPLATE.md`
- `templates/PROJECT_DESIGN_SYSTEM_TEMPLATE.md`
- `templates/SCREEN_CATALOG_TEMPLATE.md`
- `templates/RESPONSIVE_MATRIX_TEMPLATE.md`
- `templates/UI_ACCEPTANCE_CHECKLIST_TEMPLATE.md`
- `PROJECT_ARCHITECTURE.md`
- `PROJECT_VERIFICATION_PROFILE.md`
- `ROADMAP.md`
- `FEATURE_UNIT_SPEC.md` per Feature Unit
- `COMPONENT_WORK_SPEC.md` per Component Work
- `CONTEXT_PACKET.md` per Agent Run

The first four documents define the project baseline.

## 6. Human Approval

Project constraints are not active until a human approves them.

Human approval must confirm:

- the product identity is correct
- the component map is correct
- the design direction is correct enough for first execution
- the architecture constraints are not over-specified
- verification expectations are realistic
- forbidden behavior is clear

For a UI-bearing Project, approval additionally confirms the active Design
Contract version, visual references or approved direction, responsive matrix,
and required manual visual checks. A backend-only Project records
`design_applicability: not_applicable` with a reason instead of fabricating UI
documents.

## 7. Context Packet Inclusion

Agents should not receive every project constraint every time.

ContextPacket generation must include:

- global ADO execution rules for the role
- project constraints relevant to the target component
- feature unit acceptance criteria
- component work contract
- allowed paths and commands
- required output schema

ContextPacket generation must omit:

- unrelated component details
- old roadmap material not needed for the task
- stale project constraints
- rejected constraint drafts
- secrets and raw credentials

## 8. Constraint Drift

When a project changes direction, ADO creates a new profile version.

Existing active work keeps its original constraint hash unless a human explicitly rebinds it.

Rebinding requires:

- reason
- affected Feature Units
- affected Component Work
- impact note
- audit event

## 9. Service Types

Project constraints must support many service types:

- mobile app
- web app
- backend API
- admin console
- infra
- data pipeline
- design asset
- documentation
- research
- mixed product with multiple components

No template may assume that the project is only one app or only one website.

## 10. Completion Standard

Project constraint generation is complete when:

- questionnaire is answered
- generated docs exist
- generated docs reference the active ProjectConstraintProfile
- human approval is recorded
- context packet generation can select relevant excerpts
- stale/rejected versions are blocked from agent input

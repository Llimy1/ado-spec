# Agent Development Orchestrator

This directory is the canonical specification library for Agent Development Orchestrator (ADO).

ADO is a generic development orchestration system. It is not tied to a single product. Each real product project should derive its own project-level documents from these base documents.

This repository is the **ADO Spec Library**, not the ADO application monorepo.
The NestJS implementation lives in a separate `ado-platform` repository and
pins an approved immutable Spec Library revision.

## Canonical Documents

- `ADO_MASTER_SPEC.md`: compressed master specification.
- `ADO_MASTER_SPEC.ko.md`: Korean human-readable guide. The English master spec remains canonical.
- `DOCUMENT_CONSTRAINTS.md`: document-level behavior constraint rules for agents.
- `AGENT_ROLE_SPECS.md`: role-level contracts for ADO actors and agents.
- `AGENT_INGEST_PROTOCOL.md`: scoped Ingest API contract for human-started external agent results.
- `SCHEMA_CONSTRAINTS.md`: schema-level structured output rules for agents and runners.
- `RUNTIME_CONSTRAINTS.md`: runtime-level enforcement rules for agent processes and runners.
- `RUNTIME_RULES.md`: practical runtime defaults for Worker, Codex, verification, local models, and PR manager.
- `REPOSITORY_RULES.md`: git, branch, worktree, PR, and path constraints.
- `SECURITY_POLICY.md`: secret, PII, production, network, external transfer, and supply-chain policy.
- `POLICY_STATE_CONSTRAINTS.md`: Policy Engine, Evidence Gate, and State Machine constraints.
- `STATE_TRANSITION_RULES.md`: allowed state transitions and required transition rules.
- `EVIDENCE_GATES.md`: required evidence for important state transitions.
- `CODING_STANDARDS.md`: implementation coding rules for ADO itself.
- `NESTJS_MONOREPO_ARCHITECTURE.md`: canonical NestJS, TypeORM, PostgreSQL, pnpm, and Turborepo architecture for ADO itself.
- `CONTROL_ROOM_API_UI_SPEC.md`: REST/OpenAPI/SSE and Next.js control-room contract.
- `CONTROL_ROOM_DESIGN_SYSTEM.md`: visual, responsive, interaction, accessibility, and QA contract for the ADO Control Room only.
- `CONTROL_ROOM_PAGE_SPECS.md`: route-by-route API, state, UI, accessibility, responsive, and verification contracts for the Control Room.
- `BOOTSTRAP_PROTOCOL.md`: bootstrap stages plus A1 entry and acceptance criteria.
- `SPEC_LIBRARY_PLATFORM_BOUNDARY.md`: authority, versioning, and pinning contract between the Spec Library and ADO Platform.
- `MANAGED_PROJECT_MONOREPO_POLICY.md`: public managed-Project monorepo, coordinated work, branch, PR, and verification rules.
- `DJANGO_ARCHITECTURE.md`: historical Django draft only; it is not a current implementation contract.
- `SERVICE_LAYER_RULES.md`: service, selector, policy, runner, and state-machine boundaries.
- `TESTING_STRATEGY.md`: required test layers and safety coverage for ADO.
- `WORKER_EXECUTION_CONTRACT.md`: worker lifecycle, lease, outbox, runner, timeout, and recovery contract.
- `JOB_HANDLER_CATALOG.md`: enabled Job types and their handler-level contracts.
- `WORKER_OPERATIONS.md`: Worker commands, health, shutdown, recovery, and control-room operations.
- `DB_MODEL_SPEC.md`: canonical TypeORM/PostgreSQL table, field, and relationship contract.
- `DATABASE_CONSTRAINTS.md`: PostgreSQL constraints, locking, indexes, migrations, and append-only protection.
- `DATA_LIFECYCLE.md`: data creation, validation, retention, recovery, and deletion rules.
- `DATABASE_ERD.md`: relationship map for the canonical database contract.
- `PROJECT_CONSTRAINT_GENERATION.md`: how ADO creates project-specific service constraints.
- `PROJECT_DESIGN_GOVERNANCE.md`: separation, lifecycle, approval, and versioning rules for per-Project design systems.
- `SERVICE_CONSTRAINT_QUESTIONNAIRE.md`: intake questions for generating project constraints.
- `PROJECT_DERIVATION_GUIDE.md`: how to create project-specific ADO documents.
- `templates/`: reusable project, roadmap, feature unit, component work, packet, review, PR, and audit templates.
- `schemas/`: JSON schema drafts for agent outputs.

The schema directory also contains `spec_library_manifest.schema.json` and
`ado_spec_lock.schema.json` for the Spec Library/Platform integrity boundary.

## Core Principle

The database is the single source of truth. Markdown files are generated or imported working artifacts.

Document instructions guide agents, but the ADO harness enforces boundaries through schema validation, runtime sandboxing, policy checks, state transitions, and audit logs.

## Project Derivation

For each product or service, create a project-specific ADO workspace from this standard:

```text
Project -> Roadmap -> Feature Unit -> Component Work -> Agent Run
```

The master spec remains stable. Project-specific documents may differ by repositories, components, verification profiles, policies, and roadmaps.

Project-specific design, architecture, and verification constraints are generated from a human-approved ProjectConstraintProfile. They are not hardcoded into the ADO master spec.

ADO's Control Room design system is independent from every managed Project
design system. Visual rules never cross that boundary; only the shared quality
baseline is common.

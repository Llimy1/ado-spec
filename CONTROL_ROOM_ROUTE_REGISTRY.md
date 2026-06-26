# ADO Control Room Route Registry

This document is the canonical route registry for the ADO Control Room. It
turns `CONTROL_ROOM_PAGE_SPECS.md` into an implementation planning table.

The Control Room currently has:

- 10 page-spec groups;
- 19 concrete routes;
- 6 delivery slices.

Routes listed here are the only v1 Control Room routes unless this document
and `CONTROL_ROOM_PAGE_SPECS.md` are updated together.

## 1. Route Rules

1. Every route is rendered by `apps/control`.
2. Every route receives data only through typed API contracts from `apps/api`.
3. The browser never reads the database, filesystem, raw worktree, raw provider
   payload, raw artifact storage, secret, deployment credential, or production
   data directly.
4. SSE is a freshness signal only. REST snapshots remain authoritative.
5. Route parameters are decoded, schema-validated, and never interpolated into
   SQL, shell commands, filesystem paths, or provider calls.
6. Unauthorized scoped resources return non-disclosing access results. A route
   must not reveal whether a hidden cross-Project resource exists.
7. A page may render a state-changing command only when its backend command,
   policy decision, evidence gate, audit event, idempotency, and error contract
   exist.
8. `main` is protected outside ADO control. No Control Room route exposes merge
   or production deployment commands in v1.

## 2. Canonical Routes

| No. | Route | Page group | Delivery | Command surface | Primary question |
|---:|---|---|---|---|---|
| 1 | `/projects` | P-01 Projects | A2 | read-only | Which Project needs human attention now? |
| 2 | `/projects/{projectKey}` | P-02 Project Overview | A2 | read-only | Is this Project advancing safely? |
| 3 | `/projects/{projectKey}/roadmaps` | P-03 Roadmaps | A3 | read-only | Which plan governs this Project? |
| 4 | `/projects/{projectKey}/roadmaps/{roadmapKey}` | P-03 Roadmap Detail | A3 | human planning decisions | Is this Roadmap valid and decomposed safely? |
| 5 | `/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}` | P-04 Feature Unit | A3 | human planning decisions | Is this functional goal approved and gate-ready? |
| 6 | `/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}` | P-05 Component Work | A4 | execution commands | Is this Work safe to execute, verify, review, or PR? |
| 7 | `/runs/{jobAttemptId}` | P-06 Run Detail And Logs | A4 | read-only | What happened in this runner attempt? |
| 8 | `/verification-runs/{verificationRunId}` | P-07 Verification Evidence | A5 | read-only | Did deterministic verification pass against the exact Work? |
| 9 | `/reviews/{reviewGroupId}` | P-07 Review Evidence | A5 | read-only | What did the review council find and decide? |
| 10 | `/pull-requests/{pullRequestId}` | P-08 Pull Request | A5 | read-only | Was this PR created from the correct Work and evidence? |
| 11 | `/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/human-verification` | P-08 Human Verification | A5 | human verification decisions | Has the human verified the Feature Unit checklist? |
| 12 | `/decisions` | P-09 Decision Inbox | A6 | scoped human decisions | Which human decisions are waiting globally? |
| 13 | `/projects/{projectKey}/decisions` | P-09 Project Decisions | A6 | scoped human decisions | Which human decisions are waiting for this Project? |
| 14 | `/projects/{projectKey}/decisions/{decisionKey}` | P-09 Decision Detail | A6 | one human decision | What exactly is being approved, rejected, or deferred? |
| 15 | `/incidents` | P-09 Incidents | A6 | incident triage | Which incidents affect ADO operation? |
| 16 | `/projects/{projectKey}/incidents/{incidentKey}` | P-09 Incident Detail | A6 | incident triage | What is blocked and how can it be released? |
| 17 | `/projects/{projectKey}/artifacts/{artifactKey}` | P-10 Artifact Detail | A6 | read-only | What artifact is this and is it safe/valid evidence? |
| 18 | `/system` | P-10 System Health | A1 | read-only | Is the ADO platform itself reachable and healthy? |
| 19 | `/settings` | P-10 Settings | A6 | read-only in v1 | What policy/configuration is active? |

## 3. Page Groups

| Group | Concrete routes | Detail-spec file to create before implementation |
|---|---:|---|
| P-01 Projects | 1 | `CONTROL_ROOM_PAGE_DETAIL_SPECS/P-01_PROJECTS.md` |
| P-02 Project Overview | 1 | `CONTROL_ROOM_PAGE_DETAIL_SPECS/P-02_PROJECT_OVERVIEW.md` |
| P-03 Roadmaps | 2 | `CONTROL_ROOM_PAGE_DETAIL_SPECS/P-03_ROADMAPS.md` |
| P-04 Feature Unit | 1 | `CONTROL_ROOM_PAGE_DETAIL_SPECS/P-04_FEATURE_UNIT.md` |
| P-05 Component Work | 1 | `CONTROL_ROOM_PAGE_DETAIL_SPECS/P-05_COMPONENT_WORK.md` |
| P-06 Run Detail And Logs | 1 | `CONTROL_ROOM_PAGE_DETAIL_SPECS/P-06_RUN_DETAIL_LOGS.md` |
| P-07 Verification And Review Evidence | 2 | `CONTROL_ROOM_PAGE_DETAIL_SPECS/P-07_VERIFICATION_REVIEW_EVIDENCE.md` |
| P-08 Pull Request And Human Verification | 2 | `CONTROL_ROOM_PAGE_DETAIL_SPECS/P-08_PR_HUMAN_VERIFICATION.md` |
| P-09 Decision Inbox And Incidents | 5 | `CONTROL_ROOM_PAGE_DETAIL_SPECS/P-09_DECISIONS_INCIDENTS.md` |
| P-10 Artifacts, System Health, And Settings | 3 | `CONTROL_ROOM_PAGE_DETAIL_SPECS/P-10_ARTIFACTS_SYSTEM_SETTINGS.md` |

Created page detail specs are implementation contracts. Planned detail specs
must be created before implementing their routes. Detail specs must not weaken
`CONTROL_ROOM_PAGE_SPECS.md`, `CONTROL_ROOM_COMPONENT_SPECS.md`, or this route
registry.

## 4. Delivery Slices

| Slice | Routes | Minimum backend proof |
|---|---|---|
| A1 Foundation | 18 | `/v1/health`, shell layout, no browser DB/secret access |
| A2 Project read path | 1, 2 | Project tables, overview projection, read DTOs |
| A3 Planning path | 3, 4, 5 | Roadmap/Feature Unit records, planning decisions, dependency gates |
| A4 Execution path | 6, 7 | Component Work, Jobs/Attempts, worktree policy, redacted log cursor |
| A5 Evidence delivery | 8, 9, 10, 11 | Verification, review, PR, human verification evidence |
| A6 Operations | 12, 13, 14, 15, 16, 17, 19 | decisions, incidents, artifacts, read-only settings, policy visibility |

Implementation can deliver read-only route skeletons earlier only when they
show honest unavailable/empty states and do not fabricate domain data.

## 5. Command Classification

| Class | Routes | Rule |
|---|---|---|
| read-only | 1, 2, 3, 7, 8, 9, 10, 17, 18, 19 | may refresh snapshots, copy safe values, and navigate only |
| human decision | 4, 5, 11, 12, 13, 14 | must create HumanDecision-backed records through API |
| execution command | 6 | must enqueue/request through policy, evidence, and state machine; cannot run Worker directly |
| incident operation | 15, 16 | must write durable incident/pause/recovery records |

No command class includes merge, deploy, production data access, arbitrary
shell, raw prompt editing, raw provider payload exposure, or direct state
mutation.

## 6. Shared Route Parameters

| Parameter | Meaning | Validation |
|---|---|---|
| `projectKey` | immutable Project human key | lowercase or configured Project key format; scoped authorization required |
| `roadmapKey` | immutable Roadmap key within Project | resolved only under the Project |
| `featureUnitKey` | immutable Feature Unit key within Roadmap | resolved only through Project and Roadmap |
| `componentWorkKey` | immutable Component Work key within Feature Unit | resolved only through Project, Roadmap, and Feature Unit |
| `jobAttemptId` | global UUID JobAttempt ID | UUID; route is read-only and links back to scoped Work when permitted |
| `verificationRunId` | global UUID VerificationRun ID | UUID; must not reveal hidden Work |
| `reviewGroupId` | global UUID ReviewGroup ID | UUID; must not reveal hidden Work |
| `pullRequestId` | global UUID PullRequest ID | UUID; no merge command |
| `decisionKey` | human decision key within Project | resolved only under Project |
| `incidentKey` | incident key within Project | resolved only under Project |
| `artifactKey` | artifact key within Project | resolved only under Project and redaction policy |

## 7. Required Detail Spec Sections

Every file in `CONTROL_ROOM_PAGE_DETAIL_SPECS/` must include:

1. exact routes and route parameters;
2. API endpoints and DTO names;
3. query/search/pagination behavior;
4. component tree;
5. loading, empty, filtered-empty, stale, disconnected, error, denied, and
   conflict states;
6. command eligibility, disabled reason, confirmation, accepted/rejected, 409,
   and 422 behavior where commands exist;
7. desktop/tablet/mobile layout rules;
8. keyboard, focus, live-region, and reduced-motion behavior;
9. Playwright scenarios and required viewport evidence;
10. mock data fixtures required before UI implementation.

## 8. Change Control

Adding, removing, renaming, or changing the command class of a route requires:

1. update this document;
2. update `CONTROL_ROOM_PAGE_SPECS.md`;
3. update affected OpenAPI/API contracts;
4. update delivery slice evidence;
5. record the change in the relevant Feature Unit or spec PR.

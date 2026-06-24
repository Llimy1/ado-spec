# ADO Control Room API And UI Specification

This document defines the v1 control plane for Agent Development Orchestrator
(ADO). It applies to `apps/api` and `apps/control` in the NestJS monorepo.

Its visual, responsive, interaction, and accessibility token contract is
defined by `CONTROL_ROOM_DESIGN_SYSTEM.md`. That design system applies only to
the ADO Control Room, never to managed Project products.

The control room makes the database-backed orchestration system observable and
human-operable. It is not an alternate source of truth, an agent shell, or a
database administration console.

## 1. Non-Negotiable Principles

1. PostgreSQL remains the single source of truth. Every screen is a projection
   of API reads from that truth.
2. The browser never has database credentials and never writes database tables
   directly.
3. A user action creates a typed command or TransitionRequest. The API invokes
   application use cases, which apply policy, evidence gates, and the state
   machine before committing state.
4. SSE is a best-effort freshness signal. A REST snapshot is authoritative
   after reconnect, missed events, or a version conflict.
5. Raw agent output, credentials, and unsanitized provider payloads are never
   streamed to a browser. Logs and artifacts are redacted server-side and are
   retrieved only through authorized API endpoints.
6. The UI must make blocked, stale, failed, paused, and human-required work
   visible. It must not present an inferred "success" state.

## 2. Product Scope

The v1 control room supports one authenticated Human Owner operating ADO. Its
scope is operational control and evidence review:

- inspect Project, Roadmap, Feature Unit, Component Work, Job, Agent Run,
  verification, review, Git, and incident state;
- approve or reject human gates with an explicit decision record;
- submit allowed commands such as pause, resume, retry, cancel, and create
  an approved next action;
- inspect evidence, redacted logs, generated documents, review findings, and
  PR metadata;
- surface a human verification checklist after PR creation.

It does not provide production database administration, secret management,
arbitrary terminal execution, direct prompt editing for a running job, Git
merge, or unrestricted manual state changes.

## 3. Information Architecture

The navigation is project-first. A global header contains project selection,
system health, live-connection state, and the current authenticated operator.

| View | Primary question answered | Required content |
|---|---|---|
| Projects | What work exists and needs attention? | project state, active Feature Unit, blockers, latest run, open human gates |
| Project Overview | Is this project moving safely? | roadmap progress, current Feature Unit, Component Work matrix, worker health, recent audit events |
| Roadmap / Feature Unit | What is the approved functional goal and its dependency state? | source roadmap, scope, acceptance criteria, dependency graph, approval history, linked Component Work |
| Component Work | What implementation unit is executing and on which branch? | component root, allowed paths, branch/worktree, state timeline, jobs, verification, reviews, PR |
| Run Detail | What happened during this execution? | immutable attempt summary, timestamps, runner identity, redacted logs, artifacts, exit/timeout result, retry history |
| Review And Verification | Is the work ready for a PR and human verification? | deterministic verification evidence, local reviewer findings, arbiter result, resolution history, human checklist |
| Incidents | What requires recovery or a decision? | incident record, severity, affected subjects, recovery actions, audit timeline |

There is no generic editable "status" field. State is displayed from the
StateSubject projection and is changed only by named commands that explain the
requested transition.

## 4. Required UI States And Interactions

Every list, detail, and command surface implements loading, empty, error,
permission-denied, stale, and disconnected states. Data-affecting actions use
an explicit confirmation dialog when they pause, cancel, retry, or reject work.

### 4.1 State Timeline

Feature Unit and Component Work details display an append-only timeline made
from StateTransition and AuditEvent records. Each entry includes the actor,
timestamp, transition rule, PolicyDecision, relevant EvidenceGate result, and
a link to the supporting artifact where applicable.

### 4.2 Command Controls

Controls appear only when the API reports an action as allowed. The UI sends a
command with a reason and an `Idempotency-Key`; it never computes eligibility
from a local state enum. A command response is shown as `accepted`, `rejected`,
or `conflicted`, with the returned explanation and audit reference.

### 4.3 Logs And Artifacts

Run logs use a paginated, cursor-based read endpoint. New-log SSE events only
invalidate the relevant page or show a "new output available" affordance.
Large payloads are stored as Artifact metadata and fetched through an
authorized, redacted download/view endpoint. The browser must not retain a
permanent raw log copy in local storage.

### 4.4 Human Verification

At `human_verification_pending`, the UI renders the Feature Unit checklist and
Component Work-specific evidence. A Human Owner records each item as passed,
failed, or not-applicable with notes. A final close request remains a state
machine command; marking visual checkboxes alone has no state effect.

## 5. REST API Contract

All endpoints are namespaced under `/v1`. JSON uses stable, lowercase machine
state values and RFC 3339 timestamps. Responses include `requestId`; mutable
read models additionally include a version or `updatedAt` suitable for stale
data detection.

### 5.1 Read Endpoints

Read endpoints are side-effect free and cursor-paginate unbounded collections.
Keys are not assumed globally unique unless the database contract says they are.
`project_key` is global; `roadmap_key` is unique within a Project;
`feature_unit_key` is unique within a Roadmap; `component_work_key` is unique
within a Feature Unit. Routes for those human-readable keys are therefore
hierarchical. Resources that have no globally unique human key use their UUID
with an `{resourceId}` parameter. A route never treats a locally unique key as
a global identifier.

Representative endpoint families are:

```text
GET /v1/projects
GET /v1/events
GET /v1/projects/{projectKey}
GET /v1/projects/{projectKey}/roadmaps
GET /v1/projects/{projectKey}/overview
GET /v1/projects/{projectKey}/roadmaps/{roadmapKey}
GET /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units
GET /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}
GET /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}
GET /v1/projects/{projectKey}/jobs/{jobKey}
GET /v1/projects/{projectKey}/jobs/{jobKey}/attempts
GET /v1/job-attempts/{jobAttemptId}
GET /v1/agent-runs/{agentRunId}
GET /v1/verification-runs/{verificationRunId}
GET /v1/review-groups/{reviewGroupId}
GET /v1/pull-requests/{pullRequestId}
GET /v1/incidents
GET /v1/projects/{projectKey}/incidents/{incidentKey}
GET /v1/projects/{projectKey}/artifacts/{artifactKey}
GET /v1/logs?attemptId={jobAttemptId}&cursor={cursor}
GET /v1/health
```

Detail endpoints return linked identifiers and HAL-like `links` only when the
target is permitted. They do not embed every log line or artifact payload.

### 5.2 Command Endpoints

Commands use `POST` and accept an `Idempotency-Key` header. They validate the
request schema, actor authorization, policy, evidence, and expected resource
version before changing state. Long-running work returns `202 Accepted` with a
Job or TransitionRequest reference; it never holds an HTTP connection until a
runner finishes.

```text
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/commands/record-human-decision
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/commands/record-human-decision
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}/commands/pause
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}/commands/resume
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}/commands/retry
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}/commands/cancel
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}/commands/request-pr
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/human-verification-items/{itemKey}/commands/record-result
POST /v1/projects/{projectKey}/incidents/{incidentKey}/commands/acknowledge
POST /v1/projects/{projectKey}/incidents/{incidentKey}/commands/request-recovery
```

The API never offers `POST /state`, bulk update, arbitrary retry, arbitrary
shell, direct document overwrite, or merge endpoints.

### 5.3 Response And Error Semantics

Use these response classes consistently:

| Status | Meaning |
|---|---|
| `200` | completed read or synchronously completed safe command |
| `201` | newly created durable resource such as a draft or request |
| `202` | durable command accepted and queued for later execution |
| `400` | malformed payload or unsupported value |
| `401` / `403` | unauthenticated / authenticated but not permitted |
| `404` | resource is absent or intentionally undiscoverable |
| `409` | stale version, active conflicting command, or idempotency conflict |
| `422` | schema-valid request rejected by policy, state, or evidence gate |
| `429` | command or stream rate limit reached |
| `500` | unexpected server error, with no sensitive detail |

Every rejected state-changing request produces an AuditEvent. `422` responses
return a stable machine code plus a human-readable explanation and links to the
relevant policy/evidence result when authorized.

## 6. OpenAPI Contract

`apps/api` generates the authoritative OpenAPI document from validated NestJS
controllers and DTOs during CI. The document is versioned with the API and
published as a build artifact. `packages/contracts` owns shared transport types
and is regenerated or checked against the OpenAPI document in CI.

Breaking changes require a new `/v2` or an explicitly approved compatibility
plan. The control app consumes generated types or a checked client, not copied
handwritten request/response interfaces.

## 7. SSE Contract

The API exposes two authenticated, read-only streams:

```text
GET /v1/events
GET /v1/projects/{projectKey}/events
```

`GET /v1/events` is the Human Owner's global Control Room stream. It exists so
the Projects list and the Human Decision Inbox can receive attention-level
freshness signals across Projects. It emits only these summary event types:

```text
project.attention.changed
project.archived.changed
human_decision.required
incident.updated
```

It never contains log content, artifact content, provider payloads, command
arguments, or per-run output. `GET /v1/projects/{projectKey}/events` remains
the detailed stream for one authorized Project and may emit the broader event
set defined below. Both streams are hints; their associated REST snapshots are
authoritative.

The event envelope is transport-safe and contains no secret or raw payload:

```json
{
  "id": "evt_...",
  "type": "project.attention.changed",
  "occurredAt": "2026-06-22T12:00:00Z",
  "projectKey": "ado",
  "traceId": "trc_...",
  "resource": { "type": "component_work", "key": "cw_..." },
  "summary": { "state": "verification_running" },
  "href": "/v1/component-works/cw_..."
}
```

Allowed event types are:

```text
state.transitioned
project.attention.changed
project.archived.changed
job.attempt.updated
worker.updated
log.available
artifact.available
verification.completed
review.finding.created
pull_request.updated
incident.updated
pause.updated
human_decision.required
outbox.updated
```

The server derives events from committed outbox/audit records. It does not send
an SSE event before the underlying transaction commits. Event IDs support
`Last-Event-ID` best-effort resume, but the server may require a full REST
snapshot when retention has expired. The client reconnects with bounded
backoff, marks displayed data stale while disconnected, then refetches the
relevant authoritative resource after reconnection.

SSE is one-way. Browser commands always use REST. The stream is an update hint,
not a durable log transport, job queue, or state authority.

## 8. Authentication And Authorization

v1 supports one Human Owner account. The Control app and API are same-origin in
the initial deployment and use secure, HTTP-only session cookies with CSRF
protection on state-changing requests. The session is validated by API guards;
the Worker has no browser session and has a separate process identity.

Authorization is enforced in application services as well as at HTTP routing.
Every command records the authenticated actor and optional reason. Provider
credentials remain Worker-only and are never exposed through UI configuration,
OpenAPI examples, logs, errors, or SSE.

Future multi-user roles require an explicit role/permission model and audit
migration; the UI must not simulate roles before the backend enforces them.

## 9. Observability And Accessibility

The API attaches `requestId` and `traceId` to command responses, read errors,
AuditEvents, JobAttempts, and runner logs. The UI links every visible status to
the record that proves it. Health endpoints report process readiness without
exposing topology, credentials, or internal stack traces.

The control room supports keyboard navigation, clear focus indication,
semantic controls, text equivalents for state color, and responsive layouts.
Operational density is preferred over decorative dashboard treatment.

## 10. Required Tests

- controller contract tests validate DTOs, authentication, idempotency, and
  OpenAPI generation;
- application tests prove direct UI/API calls cannot bypass policy, evidence,
  or the state machine;
- integration tests cover command transaction plus outbox publication;
- SSE tests prove no event is emitted for a rolled-back transaction and that
  reconnect results in an authoritative refetch path;
- UI tests cover loading, empty, stale, disconnected, rejected-command, and
  human-verification flows;
- security tests prove redacted logs/artifacts and Worker-only credentials are
  not included in API or SSE responses.

## 11. Acceptance Criteria

This specification is implemented for an initial vertical slice only when:

1. a Human Owner can see a Project overview and a Component Work timeline from
   database-backed API reads;
2. a permitted pause/resume or human-decision command is durable, idempotent,
   audited, and policy/state-machine governed;
3. the control UI receives an SSE update after commit, reconnects safely, and
   refetches REST data when it cannot trust stream continuity;
4. OpenAPI describes every exposed REST endpoint and CI detects contract drift;
5. the UI has no database connection, no runner/provider credential, and no
   direct-state-edit capability.

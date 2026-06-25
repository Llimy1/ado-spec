# ADO Agent Ingest Protocol

This document defines how a human-started external or semi-automatic agent
submits results back into ADO.

The primary v1 use case is Claude Code interactive work. The same protocol may
later be used by other external advisors, but no provider is enabled until it
has an explicit RoleSpec, schema, policy gate, and audit path.

## 1. Purpose

ADO may ask a human to run Claude Code interactively because Claude SDK/API/MCP
automation is not part of the v1 execution path. Human-started does not mean
manual copy-back. When the external agent can make an HTTP request, it submits
the result to ADO through a scoped Ingest API.

The API request itself is the durable "agent result produced" signal.

```text
ADO creates AgentIngestRun packet
-> Human starts Claude Code with the packet
-> Claude Code produces a result
-> Claude Code POSTs the result to ADO Ingest API
-> ADO validates, stores, audits, and creates follow-up work
-> Worker observes the created Job and runs Codex when policy allows
```

## 2. Non-Negotiable Rules

1. External agents never receive direct database credentials.
2. External agents submit only through ADO Ingest API.
3. The Ingest API request is the completion event for that external run.
4. Raw submitted content is preserved as an Artifact even when structured
   validation fails, unless policy requires quarantine.
5. Structured payloads are validated against a committed schema before they
   can create automatable follow-up Jobs.
6. A submitted Claude result is a claim, not final evidence of correctness.
7. The API creates follow-up Jobs only inside the same transaction that stores
   the accepted result and audit facts.
8. Worker execution is triggered from database Job state. HTTP request arrival
   wakes the system, but DB state remains authoritative.
9. ADO UI must show the raw response, parsed result, validation status,
   generated follow-up Jobs, and any human-required reason.

## 3. Claude Code v1 Flow

### 3.1 Packet Creation

ADO creates an external-safe packet containing:

- project, roadmap, Feature Unit, and Component Work identifiers;
- the exact question/task for Claude;
- allowed context and redaction summary;
- expected `result_type`;
- `agent_ingest_run_id`;
- submit endpoint URL;
- one-time or short-lived submit token;
- idempotency key guidance;
- required result schema reference.

The packet must not include secrets, production data, PII, raw credentials, or
unredacted private payloads.

### 3.2 Human Role

The human starts Claude Code with the packet and grants any local tool
permissions required by that interactive session. The human does not translate
Claude's response into ADO state and does not manually edit the submitted
result unless the submit path fails.

### 3.3 Claude Completion Protocol

The packet instructs Claude Code to write a JSON result file and submit it:

```bash
curl -X POST "$ADO_INGEST_URL" \
  -H "Authorization: Bearer $ADO_INGEST_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $ADO_INGEST_IDEMPOTENCY_KEY" \
  --data-binary @claude-result.json
```

An `adoctl submit-agent-result` wrapper may replace raw `curl`, but it must
use the same API contract.

Claude Code must not ask the human to paste its response back into ADO unless
the submit request fails and the packet explicitly allows manual fallback.

## 4. API Contract

Endpoint:

```http
POST /v1/agent-ingest/runs/{agentIngestRunId}/result
Authorization: Bearer <one-time-or-short-lived-submit-token>
Content-Type: application/json
Idempotency-Key: <stable-uuid-or-packet-key>
```

The request body validates against:

```text
schemas/agent_ingest_result.schema.json
```

The route is not a browser command endpoint. It uses a scoped submit token,
not a Human Owner session cookie. The token is bound to one
`agent_ingest_run_id`, one Project, one expected agent kind, one expected
result type, an expiry, and one redaction boundary.

### 4.1 Accepted Response

On accepted ingest:

```http
202 Accepted
```

Response body includes:

- `agentIngestRunId`;
- accepted raw Artifact id/key;
- structured Artifact id/key when validation passes;
- validation status;
- created Job ids/keys;
- whether Codex execution is queued or waiting for human approval.

### 4.2 Rejection Response

Malformed, expired, conflicting, or policy-denied submissions return `400`,
`401`, `403`, `409`, or `422` according to
`CONTROL_ROOM_API_UI_SPEC.md`. Rejected submissions still produce an
AuditEvent when the run can be identified. Safe raw content may be retained as
quarantined evidence when policy permits.

## 5. Transaction Contract

The Ingest API handles an accepted result in one short transaction:

1. lock `AgentIngestRun` by id;
2. validate submit token, expiry, expected agent kind, and idempotency key;
3. persist raw response Artifact;
4. validate structured payload against schema;
5. persist structured result Artifact or validation failure Artifact;
6. update `AgentIngestRun` status through the StateMachine;
7. create `ArtifactSourceRef` edges to the source packet and external transfer;
8. create AuditEvent records:
   - `agent_ingest.result_received`
   - `agent_ingest.result_validated` or `agent_ingest.result_rejected`
9. create follow-up Job rows only when policy permits;
10. create JobOutboxEvent or PostgreSQL notification wake-up after commit.

External model calls, Codex execution, Git commands, and verification commands
must not run inside the ingest transaction.

## 6. Follow-Up Job Policy

The submitted result may recommend a next action, but the policy engine decides
what can be queued.

Supported v1 policies:

```text
manual_after_agent_ingest
auto_after_valid_agent_ingest
auto_only_if_low_risk
```

Default recommendation:

```text
auto_after_valid_agent_ingest
```

Automatic Codex job creation must stop when any of these are true:

- schema validation failed;
- redaction failed;
- result declares `requires_human_decision=true`;
- any risk has `severity=high` or `severity=critical`;
- policy issues are present;
- target Feature Unit or Component Work is paused, cancelled, or blocked;
- the expected result type does not match the run record;
- idempotency conflict is detected.

When automatic execution stops, ADO creates a Human Decision Inbox item or
marks the Agent Inbox item as human-required.

## 7. Worker Wake-Up

The Worker leases Jobs from PostgreSQL. It may also listen for PostgreSQL
`LISTEN/NOTIFY` or consume JobOutboxEvents as a low-latency wake-up
optimization.

The wake-up signal is never the source of truth. If a Worker misses a
notification, its reconciliation loop must still find queued Jobs.

## 8. Control Room Requirements

ADO Control Room must expose an Agent Inbox / Agent Ingest Runs surface.

Required views:

- list of ingest runs by Project, Feature Unit, Component Work, status, and
  attention reason;
- run detail with source packet metadata;
- raw submitted response viewer with redaction classification;
- structured result viewer;
- validation results and schema errors;
- risks, questions, policy issues, and human checklist;
- recommended next action;
- created follow-up Job links;
- Codex execution policy result;
- retry/resubmit affordance when policy permits;
- audit timeline.

SSE may announce safe summary events, but raw responses are fetched only by
authorized REST artifact endpoints.

## 9. Status Values

AgentIngestRun status values:

```text
packet_created
issued_to_human
submitted
validating
validated
queued_follow_up
human_required
rejected
quarantined
expired
cancelled
failed
```

The StateMachine owns status transitions. API handlers and Workers never write
status fields directly.

## 10. Security Rules

- Submit tokens are secret credentials. They must not be logged, stored in raw
  artifacts, shown in UI after issuance, or embedded in reusable documents.
- Submit tokens are scoped to one ingest run and expire quickly.
- Replays are handled by idempotency key and content hash.
- Payload size is bounded.
- Raw submitted content is treated as untrusted input.
- The browser never receives submit tokens except when explicitly issuing a
  packet to the Human Owner for copy into an external agent session.
- Claude Code must never receive database credentials.
- Claude Code must never be instructed to call internal endpoints other than
  the scoped Ingest API.

## 11. Failure Behavior

If submission fails, Claude Code should report the HTTP status, request id,
and local result file path to the human. It must not retry indefinitely.

ADO stores identifiable failed attempts as audit facts. A retry creates a new
attempt or explicitly reuses the idempotent accepted submission. It never
silently overwrites a previous result.

## 12. Out Of Scope

- Claude SDK automatic execution.
- Claude MCP automatic execution.
- Direct DB writes from Claude Code.
- Treating Claude output as final verification evidence.
- Automatic PR merge.
- External agent access to production secrets or production data.

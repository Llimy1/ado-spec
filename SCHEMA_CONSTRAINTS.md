# ADO Schema Constraints

This document defines the schema-level constraint layer for Agent Development Orchestrator (ADO).

Schema constraints are the second control layer. They constrain what agents can claim and make outputs machine-checkable. They do not prove that the agent actually complied. ADO must still verify output claims against git, command logs, artifacts, policies, and state-machine evidence.

## 1. Design Basis

ADO schema constraints are based on:

- Codex `codex exec --output-schema`, which requests a final response conforming to a JSON Schema.
- Codex `--json`, which provides event streams for observability.
- Claude Code `--json-schema` and JSON output modes, where available, for structured output in non-interactive/print mode.
- OpenAI structured output guidance: strict schema, explicit fields, refusal/error handling, and downstream validation.
- JSON Schema 2020-12 vocabulary for required fields, enums, arrays, object shape, and `additionalProperties`.
- Harness engineering: outputs must support task state, observability, failure attribution, verification, intervention, and audit.

## 2. Non-Negotiable Principle

```text
A schema-valid output is a claim.
ADO evidence decides whether the claim is true.
```

Therefore:

- Schema validation can make output parseable.
- Schema validation cannot prove implementation correctness.
- Schema validation cannot approve a state transition.
- Schema validation cannot replace verification, review, or human decision.

## 3. Validation Layers

ADO uses two validation layers.

### 3.1 Structural Schema Validation

Checks:

- valid JSON
- required fields present
- no extra properties when `additionalProperties: false`
- enums valid
- array/object/value types valid
- nullable fields explicitly set to `null`

Failure result:

```text
AgentRun.status = failed
failure_type = schema_validation_failed
```

### 3.2 ADO Semantic Validation

Checks:

- role matches Job role
- target IDs match Job target
- artifact IDs exist
- changed files match actual git diff
- output status is allowed for current state
- failure object exists when status requires it
- `ready_for_pr` claims have required evidence
- no stale/quarantined artifacts are referenced
- no policy-denied action is requested

Failure result depends on cause:

```text
policy_denied | safety_violation | human_required | failed
```

## 4. Common Output Contract

Every model-generated output schema must include:

```text
schema_version
role
status
summary
target
artifacts
assumptions
questions
risks
policy_issues
next_recommended_action
failure
```

Use empty arrays instead of omitted arrays.

Use `null` instead of omitted nullable objects.

## 5. Target Object

Every output must identify the target.

```json
{
  "project_key": "string",
  "roadmap_key": "string or null",
  "feature_unit_key": "string or null",
  "component_work_key": "string or null",
  "job_id": "string or null",
  "agent_run_id": "string or null"
}
```

The Worker validates this against the Job context.

## 6. Artifact Reference Object

Agent outputs may reference artifacts, but the Worker persists and validates them.

```json
{
  "artifact_type": "string",
  "artifact_key": "string or null",
  "description": "string",
  "status": "draft | valid | stale | superseded | rejected | quarantined"
}
```

Agents must not fabricate persisted artifact IDs. They may propose artifact descriptions. The Worker assigns DB IDs.

## 7. Failure Object

Every schema includes a nullable failure object.

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

Failure object:

```json
{
  "failure_type": "string",
  "failure_reason": "string",
  "failure_summary": "string",
  "retryable": false,
  "requires_human": false,
  "requires_revision": false,
  "safety_related": false,
  "raw_error_artifact_key": "string or null",
  "next_recommended_action": "string"
}
```

ADO semantic validation requires `failure != null` when status is `failed`, `blocked`, `human_required`, `policy_denied`, `timed_out`, or equivalent.

## 8. Status Semantics

Statuses are role-local. They are not DB state transitions.

Examples:

```text
codex_implementer.status = succeeded
```

means:

```text
The implementer claims it completed its role.
```

It does not mean:

```text
ComponentWork.status = implementation_done
```

Only the State Machine may apply DB state transitions after Policy Engine evidence checks.

## 9. Strictness Rules

All ADO output schemas should:

- use `additionalProperties: false`
- define all required top-level fields
- prefer enums for status, role, severity, risk, and category
- use arrays for repeated data
- use nullable fields explicitly
- avoid free-form nested maps unless unavoidable
- avoid provider-specific fields in common schemas
- keep machine decisions separate from human-readable summaries

## 10. Refusal and Unclear Output Handling

If a model refuses, cannot comply, or lacks information:

- return a schema-valid object when possible
- set status to `human_required`, `blocked`, or `failed`
- populate `failure`
- include questions or limitations

If the model returns non-JSON or schema-invalid output:

- preserve raw output as restricted/raw artifact if safe
- create `schema_validation_failed`
- retry only when retry policy allows

## 11. Harness Mapping

| Harness Responsibility | Schema Requirement |
|---|---|
| Task state | target object, role, status |
| Observability | artifacts, summaries, next action |
| Failure attribution | failure object and taxonomy |
| Verification | evidence arrays and verification summaries |
| Permissions | policy_issues and safety fields |
| Intervention recording | questions, human_required, next action |
| Audit | artifact refs, target refs, schema_version |
| Context consistency | target validation against Job |

## 12. Schema Files

Canonical schema files live under `schemas/`.

Current schema set:

- `ado_spec_lock.schema.json`
- `agent_ingest_result.schema.json`
- `arbiter_decision.schema.json`
- `claude_import_output.schema.json`
- `codex_planner_output.schema.json`
- `codex_implementer_output.schema.json`
- `document_generator_output.schema.json`
- `evidence_gate_result.schema.json`
- `github_pr_manager_output.schema.json`
- `local_review_result.schema.json`
- `policy_decision.schema.json`
- `project_constraint_profile.schema.json`
- `spec_library_manifest.schema.json`
- `state_transition.schema.json`
- `transition_request.schema.json`
- `verification_runner_output.schema.json`

Current v1 schemas:

- `ado_spec_lock.schema.json`
- `agent_ingest_result.schema.json`
- `arbiter_decision.schema.json`
- `claude_import_output.schema.json`
- `codex_planner_output.schema.json`
- `codex_implementer_output.schema.json`
- `document_generator_output.schema.json`
- `evidence_gate_result.schema.json`
- `github_pr_manager_output.schema.json`
- `local_review_result.schema.json`
- `policy_decision.schema.json`
- `project_constraint_profile.schema.json`
- `spec_library_manifest.schema.json`
- `state_transition.schema.json`
- `transition_request.schema.json`
- `verification_runner_output.schema.json`

## 13. Acceptance Criteria

The schema constraint layer is complete when:

- every model/runner output has a JSON schema
- all schemas parse as valid JSON
- all schemas use strict top-level objects
- every schema includes target, artifacts, policy_issues, next action, and failure
- Worker validates structural schema
- Worker validates semantic constraints
- schema failures are recorded as AgentRun/Job failures
- raw invalid output is preserved only when safe
- schema versions are tracked

## 14. Boundary Statement

Schemas make agent claims legible.

They do not make claims true.

ADO must always verify claims through harness evidence.

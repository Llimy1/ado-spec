# ADO Runtime Rules

This document defines practical runtime defaults for ADO v1.

## 1. Default Runtime Mode

v1-alpha:

```text
Python Worker process
PostgreSQL DB queue
single worker by default
ADO-managed filesystem under ADO_DATA_DIR
```

v1-stable may add multiple Workers and remote artifact storage only when they
preserve the PostgreSQL queue, outbox, state-machine, and audit contracts.

## 2. Environment Variables

Required or recommended:

```env
ADO_DATA_DIR=~/.ado
ADO_WORKER_ID=local-worker-1
ADO_WORKER_LABELS=local,codex,ollama,github
ADO_MAX_CONCURRENT_JOBS=1
ADO_DEFAULT_JOB_TIMEOUT_SECONDS=1200
ADO_CODEX_BIN=codex
ADO_OLLAMA_BIN=ollama
ADO_GH_BIN=gh
ADO_CODEX_EPHEMERAL=true
ADO_LOCAL_REVIEW_PARALLELISM=1
```

Rules:

- Environment variable values that may contain secrets are never printed.
- Child process environments are built from an allowlist.
- Provider secrets are passed only to the process that needs them.

## 3. ADO Data Directory

Default layout:

```text
~/.ado/
  projects/
    {project_key}/
      artifacts/
      docs/
      logs/
      worktrees/
  system/
    logs/
    backups/
```

## 4. Worker Defaults

```text
heartbeat_interval_seconds = 10
stale_after_seconds = 60
offline_after_seconds = 180
default_max_retries = 1
default_retry_backoff_seconds = 30
max_auto_revision_loops = 3
```

Lease, handler, outbox, timeout, shutdown, and recovery behavior is defined in
`WORKER_EXECUTION_CONTRACT.md`. These values are defaults, not a replacement
for the Worker contract.

## 5. Timeout Defaults

| Job type | Timeout | Retry |
|---|---:|---:|
| document_generation | 60s | 2 |
| git_prepare | 60s | 1 |
| codex_planning | 20m | 1 |
| codex_implementation | 40m | 0-1 infrastructure only |
| verification | 20m | 1 transient only |
| local_review_single_model | 15m | 1 |
| arbiter_review | 20m | 1 |
| github_pr_create | 2m | 2 |
| claude_import | 30s | 0 |

Implementation/test/review failures are revision inputs, not blind retries.

## 6. Codex Runner Defaults

Planner:

```bash
codex exec --cd {planning_dir} --sandbox read-only --ask-for-approval never --output-schema {schema} --output-last-message {output} --json --ephemeral -
```

Implementer:

```bash
codex exec --cd {worktree} --sandbox workspace-write --ask-for-approval never --output-schema {schema} --output-last-message {output} --json --ephemeral -
```

Arbiter:

```bash
codex exec --cd {review_dir} --sandbox read-only --ask-for-approval never --output-schema {schema} --output-last-message {output} --json --ephemeral -
```

Forbidden defaults:

```text
danger-full-access
dangerous bypass flags
network access
global wildcard web access
```

## 7. Verification Runner Defaults

VerificationProfile commands are stored as argv arrays.

Example:

```json
["npm", "run", "typecheck"]
```

The runner must reject:

- shell strings not explicitly approved
- command substitutions
- broad shell wrappers
- deployment commands
- production database commands

## 8. Local Model Runner Defaults

Default provider:

```text
ollama
```

Default reviewers:

```text
qwen_coder
devstral
llama
```

Default execution:

```text
sequential
timeout 900s per reviewer
no repository write access
ReviewPacket only
```

## 9. GitHub PR Manager Defaults

Allowed:

- push ADO branch
- create PR to integrate
- sync PR metadata
- read check status

Forbidden:

- merge PR
- target main
- push integrate
- force push
- resolve review comments
- store raw tokens

## 10. Process Capture

Every process execution records:

- argv
- cwd
- environment profile name
- start time
- finish time
- exit code
- stdout artifact
- stderr artifact
- timeout flag
- signal, if killed

## 11. Child Process Termination

Timeout sequence:

```text
send terminate
wait grace period
send kill
record timed_out
mark owned child processes closed
```

## 12. Runtime Audit

Every Job produces at least one of:

- AgentRun
- CommandRun
- VerificationRun
- PolicyDecision
- SafetyEvent
- AuditEvent

No silent job completion is allowed.

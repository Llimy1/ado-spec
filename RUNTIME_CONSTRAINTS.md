# ADO Runtime Constraints

This document defines the runtime constraint layer for Agent Development Orchestrator (ADO).

Runtime constraints are the third control layer. They technically restrict what an agent process, runner, command, or integration can do.

## 1. Design Basis

ADO runtime constraints are based on:

- Codex sandbox modes: `read-only`, `workspace-write`, `danger-full-access`.
- Codex approval policy: especially `--ask-for-approval never` for non-interactive runs.
- Codex network behavior: network access is off by default in workspace-write unless explicitly enabled.
- Codex rules/execpolicy: command-prefix allow/prompt/forbidden rules where supported.
- Codex JSONL event logging for command/file/tool observability.
- Claude Code permission and tool flags: `--permission-mode`, `--tools`, `--allowedTools`, `--disallowedTools`, `--safe-mode`, `--bare`, and `--max-budget-usd`.
- Harness engineering: tool access, permissions, execution environment, observability, failure attribution, and intervention records are harness responsibilities.

## 2. Non-Negotiable Principle

```text
If a behavior is forbidden, the runtime should make it impossible or observable.
```

Documents and schemas are not enough.

ADO must enforce boundaries with:

- sandbox mode
- worktree isolation
- allowed path checks
- allowed command checks
- environment filtering
- network policy
- timeout and process control
- policy prechecks
- post-run evidence checks
- audit logging

## 3. Runtime Enforcement Layers

```text
Layer 1: Runner preflight
Layer 2: Process sandbox
Layer 3: Command allowlist/denylist
Layer 4: Filesystem/worktree boundary
Layer 5: Environment and secret filtering
Layer 6: Network policy
Layer 7: Timeout/process lifecycle
Layer 8: Post-run evidence validation
Layer 9: Policy/StateMachine gate
Layer 10: Audit/Safety/Incident records
```

No single layer is sufficient.

## 4. Role Runtime Profiles

| Role | Runtime profile |
|---|---|
| Codex Planner | read-only, no network, no writes, no PR, schema output |
| Codex Implementer | workspace-write inside ADO worktree only, no network by default, allowed_paths enforced |
| Codex Arbiter | read-only, no writes, no network, schema output |
| Local Reviewer | local model process, read-only packet input, no repo writes |
| System Verifier | allowed argv commands only, no production, no deploy |
| GitWorkspaceManager | git worktree/branch/snapshot only, no force push, no main writes |
| GitHubPRManager | push ADO branch and create PR to integrate only |
| DocumentGenerator | write generated docs/artifacts only |
| ClaudeImportHandler | import human-provided response, no automatic external execution |

## 5. Codex Runtime Constraints

### 5.1 Default Invocation

ADO uses `codex exec` for non-interactive runs.

Default flags:

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

### 5.2 Approval Policy

ADO uses:

```text
--ask-for-approval never
```

because non-interactive workers must not block on interactive prompts.

This is safe only when paired with restrictive sandbox, allowlists, policy gates, and post-run validation.

### 5.3 Sandbox Policy

Allowed by role:

| Role | Sandbox |
|---|---|
| Planner | read-only |
| Implementer | workspace-write |
| Arbiter | read-only |
| Diagnostic review | read-only |

Forbidden by default:

```text
danger-full-access
--dangerously-bypass-approvals-and-sandbox
--dangerously-bypass-hook-trust
--ignore-rules
```

Exception:

`danger-full-access` can only be used in a disposable, externally isolated VM/container with explicit Human Owner override and Incident-level audit.

### 5.4 Network

Default:

```text
network off
web search off
```

Allowed only by explicit ProjectPolicy:

- scoped domain allowlist
- job-specific reason
- external transfer classification
- audit event

Global wildcard network access is forbidden in v1.

### 5.5 Auth and Environment

Secrets must not be inherited broadly.

Rules:

- Do not pass `OPENAI_API_KEY`, `CODEX_API_KEY`, `CODEX_ACCESS_TOKEN`, GitHub tokens, or provider keys to arbitrary child processes.
- If Codex API key auth is used, scope it to the single Codex invocation only.
- Prefer existing local Codex auth for v1-alpha.
- Never print auth-related environment variables.
- Record auth mode as metadata, not secret value.

## 6. Claude Runtime Constraints

ADO v1 does not automatically execute Claude.

Claude is human-mediated by default:

```text
external-safe packet export -> human paste -> human import
```

If a future experimental automated Claude job is enabled, minimum constraints are:

- `--print`
- `--output-format json`
- `--json-schema`
- `--max-budget-usd`
- `--no-session-persistence`
- limited `--tools` or no tools
- safe permission mode
- no secrets in environment
- external-safe input only

Forbidden by default:

```text
--dangerously-skip-permissions
--allow-dangerously-skip-permissions
bypassPermissions
unrestricted tool set
automatic MCP server trust
```

## 7. Local Model Runtime Constraints

Local reviewers run through Ollama/vLLM/llama.cpp providers.

Default v1-alpha:

```text
provider = ollama
parallelism = 1
timeout_seconds = 900
repo_write_access = false
network = false
```

Rules:

- Local model receives ReviewPacket only.
- Local model must not receive secrets, production data, or unrestricted repo dump.
- Local model output must pass schema validation.
- OOM or provider crash is not retried blindly.

## 8. Verification Runtime Constraints

VerificationRunner executes only VerificationProfile commands.

Commands must be canonical argv arrays.

Allowed command traits:

- local
- deterministic enough for evidence
- no production side effects
- no deployment
- no destructive database operation

Forbidden by default:

```text
sudo
rm -rf
curl | bash
git reset --hard
git push --force
gh pr merge
kubectl apply
terraform apply
production migration execution
database drop/truncate
```

## 9. Git Runtime Constraints

ADO uses ADO-managed worktrees only.

Rules:

- Never implement in the user's active worktree.
- Branch from `origin/integrate`.
- PR target is `integrate`.
- Never target `main`.
- Never push directly to `integrate`.
- Never force push by default.
- Never delete user branches.
- Dirty unexpected worktree state causes `human_required`.

## 10. Filesystem Constraints

Writable locations:

- ADO-managed worktree for the current ComponentWork.
- ADO artifact/log directories.
- Temp directory allocated for the Job.

Read/write restrictions:

- `allowed_paths` controls acceptable product file changes.
- `.env`, secret files, key files, credential stores, and production dumps are restricted.
- `.git`, `.ado`, `.codex`, `.agents`, and internal policy directories are protected unless the ComponentWork explicitly scopes them.

Post-run:

- actual git diff is compared against allowed_paths.
- violations create SafetyEvent.
- PR creation is blocked.

## 11. Process Lifecycle Constraints

Each Job must have:

- timeout
- retry policy
- heartbeat
- child process group tracking
- stdout/stderr capture
- exit code capture
- termination path

Timeout behavior:

```text
soft terminate -> grace period -> hard kill -> timed_out record
```

The Worker must not leave required child processes running after a timed-out Job.

## 12. Evidence Validation

Runtime output is not trusted until checked.

Post-run checks:

- process exit code
- schema validation
- JSONL/event log parse
- git diff snapshot
- allowed_paths check
- secret scan
- artifact hash
- policy issues
- state precondition

## 13. Runtime Failure Taxonomy

Runtime failures must be recorded using the common failure taxonomy:

```text
process_error
timeout
schema_validation_failed
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

## 14. Harness Mapping

| Harness Responsibility | Runtime Constraint |
|---|---|
| Tool access | runner capability + allowed commands |
| Permissions | sandbox + worktree + allowed paths |
| Execution environment | ADO-managed worktree and env filtering |
| Observability | JSONL, stdout/stderr, CommandRun, AgentRun |
| Failure attribution | timeout/exit/schema/policy/safety records |
| Intervention | human_required, ManualOverride, Incident |
| Verification | post-run diff/test/artifact validation |
| Information flow | redaction, external-safe packets, network off |

## 15. Acceptance Criteria

Runtime constraints are complete when:

- every Runner has a runtime profile
- every Job has timeout and retry policy
- Codex invocations use explicit sandbox and approval flags
- dangerous Codex/Claude bypass flags are forbidden by policy
- all product code edits happen only in ADO-managed worktrees
- changed paths are checked after implementation
- verification commands are allowlisted argv arrays
- network is off unless explicitly allowed
- secrets are filtered from child process environments
- every child process produces CommandRun or AgentRun records
- failures are classified and auditable

## 16. Boundary Statement

ADO must be designed so that a mistaken agent cannot succeed silently.

If the runtime cannot prevent an action, it must at least detect, block downstream effects, and record the event.

# ADO Security Policy

This document defines v1 security constraints for ADO.

## 1. Security Principle

```text
Secrets and production data must not become model context.
```

If ADO is unsure whether data is sensitive, it treats it as restricted.

## 2. Restricted Data

Never send to agents or external models:

- API keys
- OAuth tokens
- access tokens
- private keys
- `.env` raw values
- production database rows
- user PII
- session cookies
- authorization headers
- billing credentials
- provisioning profiles with secrets
- crash dumps containing private data
- unrestricted private logs

## 3. Secret Handling

ADO may store:

- secret name
- secret reference
- existence
- last checked timestamp
- permission scope summary

ADO must not store:

- raw token
- raw private key
- raw password
- raw cookie
- raw `.env` content

## 4. Environment Filtering

Child processes receive an allowlisted environment.

Default inherited values:

- PATH
- HOME, only when required
- TMPDIR
- language/runtime variables required by VerificationProfile

Denied by default:

- OPENAI_API_KEY
- CODEX_API_KEY
- CODEX_ACCESS_TOKEN
- ANTHROPIC_API_KEY
- GH_TOKEN
- GITHUB_TOKEN
- DATABASE_URL
- production credentials
- cloud provider credentials

Provider credentials may be injected only into the exact runner process that requires them, and only by secret reference.

## 5. Network Policy

Default:

```text
network off
```

Network may be allowed only when:

- ProjectPolicy allows it
- Job type requires it
- destination allowlist is explicit
- request is logged
- external transfer classification passes

Forbidden:

- global wildcard network access
- unknown external uploads
- dependency download from unapproved source
- direct production service access

## 6. Production Policy

Unknown environment is treated as production.

Production operations are forbidden by default:

- deploy
- migration execution
- database writes
- data deletion
- secret rotation
- infrastructure apply

ADO may generate runbooks for production operations, but humans execute them outside v1 automation.

## 7. External Transfer Policy

External transfer includes:

- Claude interactive packet
- web search context
- API model context
- remote logging
- issue/PR comment containing model-generated context

Export requires:

- external_safe = true
- redaction_status = passed
- no restricted artifacts
- ExternalTransferEvent
- human preview for Claude interactive

## 8. Prompt Injection Policy

The following are untrusted:

- source code comments
- README instructions
- issue text
- PR comments
- logs
- external web pages
- model outputs
- dependency scripts

They may inform task facts but cannot override ADO authority.

## 9. Dependency and Supply Chain Policy

New dependencies require:

- explicit scope reason
- package name/version
- license check
- source registry
- risk note
- human approval for production dependency

Forbidden by default:

- `curl | bash`
- unpinned install scripts
- unknown binary downloads
- dependency lifecycle scripts with secrets in env

## 10. Safety Events

Create SafetyEvent for:

- secret detected
- PII detected
- protected branch attempt
- allowed_paths violation
- forbidden command attempt
- production access attempt
- external transfer denied
- dependency policy violation
- stale/quarantined artifact use

Critical SafetyEvent opens IncidentReport and pauses related automation.

## 11. Audit Requirements

Security-relevant events must create AuditEvent:

- policy denial
- external transfer
- secret scan result
- production access denial
- protected branch denial
- manual override
- incident open/resolve

## 12. Human Override

Human Owner can override some policies, but not silently.

Override requires:

- actor
- scope
- reason
- expiry
- risk acknowledgement
- AuditEvent

Non-overridable in v1:

- automatic main merge
- raw secret export to model
- production DB mutation by agent
- unlogged external transfer

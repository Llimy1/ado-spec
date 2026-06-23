# Project Verification Profile

```yaml
ado_doc_type: project_verification_profile
project_key: "{{project_key}}"
constraint_profile_id: "{{constraint_profile_id}}"
source_version: "{{source_version}}"
context_hash: "{{context_hash}}"
status: draft
stale: false
generated_from_db: true
manual_edit_detected: false
generated_at: "{{generated_at}}"
```

## 1. Verification Philosophy

- Automatic verification proves:
- Human verification proves:
- Evidence required before PR:
- Evidence required after PR:

## 2. Component Verification Matrix

| Component Key | Required Commands | Optional Commands | Human Checks | Evidence |
| --- | --- | --- | --- | --- |
|  |  |  |  |  |

## 3. Command Profiles

### 3.1 Frontend

- Install:
- Lint:
- Typecheck:
- Unit tests:
- Build:
- E2E:
- Visual:

### 3.2 Backend

- Install:
- Lint:
- Typecheck:
- Unit tests:
- Integration tests:
- Migration check:
- Build:

### 3.3 Mobile

- Install:
- Static analysis:
- Unit tests:
- Build:
- Simulator:
- Device:

## 4. Human Verification Checklist Defaults

### 4.1 Product Flow

- [ ] 
- [ ] 
- [ ] 

### 4.2 UI/UX

- [ ] 
- [ ] 
- [ ] 

### 4.3 API/Data

- [ ] 
- [ ] 
- [ ] 

### 4.4 Safety

- [ ] No secret or PII was exposed.
- [ ] No forbidden branch was modified.
- [ ] No production system was touched.

## 5. Evidence Artifacts

- Required logs:
- Required screenshots:
- Required recordings:
- Required API samples:
- Required database evidence:

## 6. Risk Levels

| Risk | Required Verification |
| --- | --- |
| low |  |
| medium |  |
| high |  |
| critical |  |

## 7. PR Gate

PR creation is allowed only when:

- required command verification passed
- blocking review findings are resolved or accepted by human override
- required evidence artifacts are valid
- human verification checklist has been generated
- branch and target follow repository rules

# Project Architecture

```yaml
ado_doc_type: project_architecture
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

## 1. Component Map

Layout profile:

```text
standard_product_monorepo
```

| Component Key | Type | Repository | Root Path | Internal Structure Profile | Runtime | Owner |
| --- | --- | --- | --- | --- | --- | --- |
|  |  |  |  |  |  |  |

## 1.1 Repository Layout

Approved root layout:

```text
apps/
packages/
docs/
assets/
infra/
tools/
tests/
.ado/
```

Omitted default roots:

-

Custom roots:

-

Generated paths:

- `.ado/generated/`
- `.ado/packets/`
- `.ado/reports/`

Human-authored documentation paths:

- `docs/product/`
- `docs/architecture/`
- `docs/verification/`
- `docs/decisions/`

## 1.2 Component Internal Structure

| Component Key | Internal Structure Profile | Required Directories | Notes |
| --- | --- | --- | --- |
| web | nextjs_feature_app | `src/app`, `src/features`, `src/components`, `src/lib` |  |
| mobile | expo_feature_app | `src/app` or `src/screens`, `src/features`, `src/components`, `src/lib` |  |
| api | nestjs_module_api | `src/modules`, `src/common`, `src/config`, `src/database` |  |
| shared | pure_shared_package | `src/domain`, `src/utils`, `src/validation`, `src/types` |  |
| ui | project_ui_package | `src/components`, `src/primitives`, `src/tokens`, `src/styles` |  |
| contracts | transport_contract_package | `src/api`, `src/events`, `src/schemas` |  |

## 2. Repository Rules

- Protected branch:
- Integration branch:
- ADO branch prefix:
- PR target:
- Forbidden paths:
- Human approval required paths:
- Generated paths:
- ADO-managed paths:

## 3. Frontend Architecture

- Framework:
- Routing:
- State management:
- Component library:
- Styling:
- API client:
- Test commands:
- Forbidden patterns:

## 4. Backend Architecture

- Framework:
- API style:
- Database:
- Auth:
- Background jobs:
- External services:
- Test commands:
- Forbidden patterns:

## 5. Mobile Architecture

- Framework:
- Platforms:
- Navigation:
- State management:
- Native permissions:
- Build commands:
- Test commands:
- Forbidden patterns:

## 6. Data Rules

- Sensitive data:
- PII:
- Retention:
- Audit:
- Migration rules:
- Backup/restore notes:

## 7. API Rules

- Request format:
- Response format:
- Error format:
- Pagination:
- Versioning:
- Authentication:
- Authorization:

## 8. Dependency Rules

- Allowed dependency changes:
- Human approval required:
- Forbidden dependencies:
- Package manager:
- Lockfile policy:

## 9. Operational Rules

- Environment variables:
- Logging:
- Monitoring:
- Feature flags:
- Rollback:
- Deployment notes:

## 10. Architecture Decisions

| Date | Decision | Reason | Human Approver |
| --- | --- | --- | --- |
|  |  |  |  |

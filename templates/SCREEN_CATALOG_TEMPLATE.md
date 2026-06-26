# Screen Catalog

```yaml
ado_doc_type: screen_catalog
project_key: "{{project_key}}"
component_key: "{{component_key}}"
constraint_profile_id: "{{constraint_profile_id}}"
design_contract_version: "{{design_contract_version}}"
context_hash: "{{context_hash}}"
status: draft
generated_from_db: true
manual_edit_detected: false
generated_at: "{{generated_at}}"
```

This document lists every UI screen and important state for one UI-bearing
Component.

## 1. Component Summary

- Component:
- Platform:
- Primary user:
- Main user goal:
- Related Feature Units:
- Design system version:

## 2. Screen Inventory

| Screen ID | Route/View | User question | Primary action | Data dependencies | Status |
|---|---|---|---|---|---|
| `screen-001` |  |  |  |  | draft |

## 3. Required States Per Screen

For each screen, define applicable states.

```text
default
loading
empty
error
permission_denied
stale
offline_or_disconnected
conflict
success_feedback
long_content
```

## 4. Screen Detail Template

### Screen: `{{screen_id}}`

- Route/view:
- Primary user:
- User question:
- Primary action:
- Secondary actions:
- Data dependencies:
- State source:
- Empty state:
- Loading state:
- Error state:
- Permission state:
- Recovery path:
- Long text/ID/path behavior:
- Accessibility notes:
- Responsive notes:
- Human verification notes:

## 5. Navigation And Flow

- Entry points:
- Exit points:
- Back behavior:
- Deep-link behavior:
- Auth/permission behavior:

## 6. Open Questions

-

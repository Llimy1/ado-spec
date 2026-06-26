# UI Acceptance Checklist

```yaml
ado_doc_type: ui_acceptance_checklist
project_key: "{{project_key}}"
component_key: "{{component_key}}"
feature_unit_key: "{{feature_unit_key}}"
component_work_key: "{{component_work_key}}"
constraint_profile_id: "{{constraint_profile_id}}"
design_contract_version: "{{design_contract_version}}"
context_hash: "{{context_hash}}"
status: draft
generated_from_db: true
manual_edit_detected: false
generated_at: "{{generated_at}}"
```

This checklist defines automated and human UI verification for one Component
Work item.

## 1. Scope

- Component Work:
- Screens changed:
- States changed:
- Design system version:
- Responsive matrix version:
- Known exceptions:

## 2. Automated Checks

- [ ] Unit/component tests cover changed UI behavior.
- [ ] Integration tests cover API/data state used by the UI.
- [ ] Browser or app smoke test covers the primary path.
- [ ] Accessibility checks cover labels, focus, contrast, and keyboard path.
- [ ] Responsive checks cover required viewport matrix.
- [ ] No console errors or runtime warnings in primary path.
- [ ] Long text, IDs, paths, and empty/error states are tested where relevant.

## 3. Visual Checks

- [ ] Layout has no unintended clipping.
- [ ] Layout has no unintended overlap.
- [ ] Page has no unintended horizontal overflow.
- [ ] Focus ring is visible and not obscured.
- [ ] Loading, empty, error, and permission states are visually distinct.
- [ ] State is not represented by color alone.
- [ ] Typography, spacing, radius, and density match the approved design system.

## 4. UX Flow Checks

- [ ] Primary user goal is reachable without hidden steps.
- [ ] Destructive or irreversible actions require appropriate confirmation.
- [ ] Error messages explain recovery.
- [ ] Disabled actions explain why they are disabled.
- [ ] Success feedback does not claim more than the backend committed.
- [ ] Back, cancel, retry, and recovery paths are clear.

## 5. Accessibility Checks

- [ ] Keyboard navigation reaches all primary controls.
- [ ] Screen-reader labels are meaningful.
- [ ] Form errors are associated with fields.
- [ ] Reduced-motion preference is respected.
- [ ] Touch targets meet the approved platform rule.

## 6. Human Verification

The human reviewer must check:

-

## 7. Evidence

| Evidence | Required | Artifact ID | Result |
|---|---|---|---|
| viewport screenshots | yes/no |  |  |
| accessibility report | yes/no |  |  |
| browser/app smoke test | yes/no |  |  |
| human review notes | yes/no |  |  |

## 8. Decision

- Result: `pass | fail | needs_revision`
- Reviewer:
- Reviewed at:
- Notes:

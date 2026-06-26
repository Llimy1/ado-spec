# ADO Control Room Component Specs

This document defines the reusable component contracts for `apps/control`. It
extends `CONTROL_ROOM_DESIGN_SYSTEM.md`, `CONTROL_ROOM_DESIGN_TOKENS.md`, and
`CONTROL_ROOM_PAGE_SPECS.md`.

## 1. Component Contract Format

Each Control Room component must define:

1. purpose;
2. allowed variants;
3. anatomy;
4. data source and state source;
5. loading, empty, error, permission, stale, disconnected, and conflict states
   where applicable;
6. keyboard and screen-reader behavior;
7. responsive behavior;
8. verification evidence.

Reusable components live under:

```text
apps/control/src/components
apps/control/src/features/<feature>/components
```

Feature-local components may not redefine global tokens.

## 2. Button

### Purpose

Trigger an API command, local navigation, copy action, or non-mutating view
change.

### Variants

| Variant | Usage |
|---|---|
| `primary` | one main safe action in a section |
| `secondary` | normal actions |
| `danger` | destructive or irreversible request |
| `ghost` | low-emphasis local UI action |
| `icon` | common utility actions with tooltip and accessible name |

### Rules

- Minimum hit target is `44 x 44px`.
- Desktop visual height is `36px`.
- Destructive, reject, cancel, pause, and retry actions require visible Korean
  text. Icon-only is forbidden for these actions.
- Disabled buttons expose the reason through adjacent text or tooltip.
- A button must show requested/accepted/rejected/conflicted state when it sends
  a state-changing command.

## 3. Status Badge

### Purpose

Display projected system state without acting as a control.

### Rules

- Badges are never buttons.
- Every badge includes icon, Korean label, and accessible text.
- Color alone is never sufficient.
- Machine value may appear in monospace in detail views.
- The badge state must come from API projection, not client inference.

Canonical labels:

```text
대기, 실행 중, 차단됨, 검토 필요, 실패, 완료, 검증 완료, 취소됨
```

## 4. Data Table

### Purpose

Display sortable or filterable operational records such as Projects, Feature
Units, Component Work, JobAttempts, Evidence, and Pull Requests.

### Anatomy

```text
toolbar
column header row
data rows
pagination or continuation affordance
empty/error/stale region
```

### Rules

- Tables are used at `768px` and above when column meaning can remain visible.
- Below `768px`, tables convert to semantic cards unless the data is a bounded
  log/code grid.
- Sticky headers are allowed only inside a bounded scroll container.
- Long IDs, paths, branches, and SHA values truncate with copy action.
- Sort and filter state is reflected in URL query where the page spec requires
  shareable state.
- New SSE data must not reorder rows automatically. The UI shows
  `새 업데이트 N건`.
- Row selection must not be the only path to a critical action.

## 5. Semantic Card List

### Purpose

Represent table-like records on narrow screens while preserving data meaning.

### Rules

- Each card has a title, status, primary metadata, and allowed actions.
- Data labels stay visible; values are not shown as unlabeled paragraphs.
- Cards do not nest inside other cards.
- A card list must preserve the same filtering and command rules as the table
  representation.

## 6. Panel

### Purpose

Group one operational topic such as state summary, evidence, queue status, or
review output.

### Rules

- Panels use `1px` border, `6px` radius, and approved padding.
- Panels are not used for every page section; full-width constrained layouts
  are preferred when framing adds no meaning.
- Panels do not nest unless the inner region is a bounded log, code, or JSON
  viewer.

## 7. Dialog

### Purpose

Collect explicit confirmation, human decision, rejection reason, conflict
resolution, or scoped command parameters.

### Rules

- Dialogs trap focus and return focus to the opener.
- Title states the action, not a generic warning.
- Body explains consequence, blocker, or policy reason.
- Dangerous actions require visible consequence copy and typed or explicit
  confirmation when specified by page contract.
- API `409` and `422` responses keep the dialog open and show recovery paths.
- Dialogs never claim completion until the API accepts and committed state is
  observed or returned.

## 8. Drawer

### Purpose

Provide mobile navigation or contextual side content below `768px`.

### Rules

- The navigation drawer is modal on mobile.
- It traps focus, closes on Escape, and labels the current route.
- It does not contain destructive commands.

## 9. Toast And Notice

### Purpose

Report transient acknowledgement or durable system notice.

### Rules

- Toasts cannot be the only record of a command result.
- Critical failures, conflicts, and blocked work must also appear in the page
  body or timeline.
- Toasts auto-dismiss only for non-critical success/info.
- Screen readers receive polite or assertive announcements according to severity.

## 10. Form Field

### Purpose

Collect filter values, decision reasons, command parameters, or configuration
inputs.

### Rules

- Every field has a visible label.
- Required fields show requirement in text, not only color or asterisk.
- Error text is adjacent to the field and linked with `aria-describedby`.
- Validation errors use backend error codes when available.
- Inputs that accept branch names, paths, keys, UUIDs, or enums show example
  format and preserve machine casing.

## 11. Tabs And Segmented Controls

### Purpose

Switch between related views of the same resource.

### Rules

- Tabs are for navigation between panels or resource subviews.
- Segmented controls are for local mode/filter changes.
- Active state is visible without relying on color.
- Keyboard operation follows expected arrow-key behavior where applicable.

## 12. Timeline

### Purpose

Display AuditEvents, state transitions, worker attempts, policy decisions, and
human decisions in temporal order.

### Rules

- The ordering key is explicit in the API response.
- Relative time is visible; absolute timestamp with time zone is available on
  hover/focus.
- Actor, action, result, and evidence link are visible when available.
- Redacted entries must state that redaction occurred without exposing content.

## 13. Log Viewer

### Purpose

Show sanitized runner, verification, or system logs.

### Rules

- Logs are plain text, not HTML.
- The server sanitizes logs before delivery.
- The viewer preserves line breaks and supports local horizontal scrolling.
- Auto-scroll is opt-in and visibly paused when the user scrolls upward.
- Copy actions copy exactly the visible sanitized content unless raw access is
  explicitly authorized by a separate policy.

## 14. JSON Viewer

### Purpose

Show structured payloads, schema validation results, context packets, and
adapter responses.

### Rules

- JSON is rendered as text or safe structured nodes, never injected HTML.
- Collapsed sections preserve enough label/context for review.
- Invalid JSON shows parser error and raw sanitized text.
- Copy action includes the full current payload.

## 15. Command Bar

### Purpose

Render allowed actions for a resource based on API policy projection.

### Rules

- The command bar displays only server-reported allowed actions.
- Blocked actions may appear only when the reason is useful and non-sensitive.
- Every command records actor, resource, expected version, and reason when
  required by API contract.
- The client never synthesizes a state transition locally.

## 16. Evidence Tile

### Purpose

Display test, screenshot, log, schema, PR, or human-verification evidence.

### Rules

- The tile shows evidence type, producer, timestamp, result, and linked subject.
- Passing evidence cannot imply overall completion unless the Evidence Gate
  says the transition is allowed.
- Missing evidence states name the required evidence and responsible actor.

## 17. Verification Requirements

Component implementation must include:

1. keyboard path coverage for primary commands;
2. disabled/rejected/conflicted command states;
3. responsive table-to-card or bounded-scroll proof;
4. loading, empty, error, stale, disconnected, and permission variants where
   applicable;
5. no visual contradiction with durable backend state;
6. visual QA screenshots at the approved matrix for affected screens.

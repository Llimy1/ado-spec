# P-01 Projects Detail Spec

Parent contract: `CONTROL_ROOM_PAGE_SPECS.md#p-01-projects`.
Route registry entry: `CONTROL_ROOM_ROUTE_REGISTRY.md` route 1.

## 1. Route Identity

| Property | Contract |
|---|---|
| Route | `/projects` |
| Page group | P-01 Projects |
| Delivery slice | A2 Project read path |
| Command class | read-only |
| Permitted actor | authenticated `Human Owner` in v1 |
| Primary question | Which Project needs human attention now, and where should the owner open it? |

This page is the global operational index. It does not create Projects, edit
Project configuration, mutate state, run Workers, show raw logs, merge PRs, or
perform cross-Project bulk commands.

## 2. Implementation Files

Target files:

```text
apps/control/src/app/(control)/projects/page.tsx
apps/control/src/features/projects/project-list-route.tsx
apps/control/src/features/projects/project-list-state.ts
apps/control/src/features/projects/project-list-query.ts
apps/control/src/features/projects/project-list-table.tsx
apps/control/src/features/projects/project-list-cards.tsx
apps/control/src/features/projects/project-filter-controls.tsx
apps/control/src/features/projects/project-attention-summary.tsx
apps/control/src/features/projects/project-list-empty-state.tsx
apps/control/src/features/events/use-global-control-events.ts
packages/contracts/src/projects/project-list.contract.ts
packages/contracts/src/events/global-control-event.contract.ts
```

`page.tsx` performs the initial no-store server snapshot and passes typed data
to `project-list-route.tsx`. Feature components may not call the database,
construct SQL, read secrets, or derive operational state from raw rows.

## 3. API Contract

Endpoint:

```text
GET /v1/projects
```

Query:

| Parameter | Type | Default | Rule |
|---|---|---|---|
| `q` | string | omitted | optional, trimmed, 1..120 chars after trim |
| `archived` | boolean | `false` | archived Project lifecycle filter |
| `attention` | repeated enum | omitted | `action_required`, `blocked`, `incident_hold` |
| `sort` | enum | `attention` | `attention`, `activity`, `name` |
| `direction` | enum | `desc` | `asc`, `desc`; default for name is `asc` |
| `cursor` | opaque string | omitted | signed/tamper-evident keyset cursor |
| `limit` | integer | `25` | 1..100 |

Invalid query returns `400` with stable machine error code. The endpoint
returns `Cache-Control: private, no-store`.

Contract files:

```text
packages/contracts/src/projects/project-list.contract.ts
packages/contracts/src/events/global-control-event.contract.ts
```

DTO names:

```text
ProjectListQuery
ProjectOperationalStatus
ProjectAttentionReason
ProjectListItem
ProjectListResponse
GlobalControlEvent
ProjectAttentionChangedEvent
```

Generated client names:

```text
getProjects(query: ProjectListQuery): Promise<ProjectListResponse>
subscribeGlobalControlEvents(options): EventSourceHandle
```

The generated OpenAPI document must expose every query parameter and every
response field. DTOs are versioned through the API contract, not through UI-only
types.

## 4. Read Model Rules

`ProjectListResponse` is produced by one bounded read model:

```text
ProjectListProjection
```

The query must not load Project rows and then perform one relation query per
row. It may use joins, subqueries, or a materialized read model.

Operational status precedence:

```text
archived
> incident_hold
> blocked
> attention_required
> healthy
```

`operationalStatus` is a projection. It is not a persisted Project state and
cannot be edited by the browser.

Required projection tests:

- archived wins over every attention state;
- incident hold wins over blocked and action-required;
- blocked wins over action-required;
- action-required appears for pending human decision, failed verification, or
  accepted revision finding;
- healthy means no current attention condition, not that all history passed.

## 5. URL And Client State

The URL is the only durable browser state. Defaults are omitted.

Example:

```text
/projects?q=orion&attention=blocked&attention=action_required&sort=attention&direction=desc
```

Client display phases:

```text
initial_loading
ready
refreshing
loading_more
error_without_data
error_with_stale_data
disconnected
```

Route reducer state:

```text
itemsByProjectKey
renderedProjectKeys
nextCursor
currentQuery
summary
observedAt
requestId
pendingUpdateCount
newestPendingEventSeverity
displayPhase
```

Search input uses a `250ms` debounce and `router.replace`. Filter changes,
filter reset, sort changes, and direction changes also use `router.replace`.
Typing does not create one browser-history entry per keystroke. "Load more" is
a button, not automatic infinite scroll.

The reducer uses request sequence IDs and `AbortController`; an older delayed
response must not replace a newer query result.

## 6. Component Tree

```text
ProjectsPage
└─ ProjectListRoute
   ├─ ProjectsHeader
   ├─ ProjectFilterControls
   │  ├─ SearchField
   │  ├─ AttentionFilterGroup
   │  ├─ ArchivedToggle
   │  └─ SortControl
   ├─ ConnectionFreshnessIndicator
   ├─ ProjectAttentionSummary
   ├─ ProjectListContent
   │  ├─ ProjectListTable
   │  └─ ProjectListCards
   ├─ LoadMoreRegion
   ├─ ProjectListEmptyState
   ├─ ProjectListErrorState
   ├─ MobileFilterDialog
   └─ LiveUpdateAnnouncer
```

Shared component mapping:

| Need | Component spec |
|---|---|
| refresh/apply/clear/load-more buttons | `Button` |
| operational status | `Status Badge` |
| desktop list | `Data Table` |
| tablet/mobile list | `Semantic Card List` |
| filter modal | `Dialog` |
| stale/error notice | `Toast And Notice` plus inline notice |
| timestamp and attention events | `Timeline` rules for time display only |

Rows are not click targets. Project names and Feature Unit names are normal
links. Sort headers are real buttons inside native table headers.

## 7. Layout Contract

Desktop `1280px+` order:

```text
breadcrumb
page title + result count + last authoritative snapshot time
search + filter controls + sort control + connection freshness
filtered-result attention summary
semantic Project table
load-more / end-of-results region
```

Responsive matrix:

| Range | Navigation | Representation | Rule |
|---|---|---|---|
| `1280px+` | 256px sidebar | full semantic table | all columns visible |
| `1024-1279px` | 256px sidebar | compact semantic table | Component Work moves into Project cell; Open column removed |
| `768-1023px` | 64px rail | two-column card list | description and secondary counts move to card detail |
| `320-767px` | Drawer | one-column card list | one highest reason, active Feature Unit, and activity visible |

No page-level horizontal overflow is allowed. Between `1024px` and `1279px`,
the table wrapper may have local horizontal scroll only when font scaling or
localization makes compact minimums impossible. The wrapper must be keyboard
scrollable and visually indicate overflow.

## 8. Text And Overflow

| Content | Rule |
|---|---|
| Project name | one line in tables, two lines in cards |
| `projectKey` | one monospace line, ellipsis, copy button, tooltip |
| description | one desktop line, hidden below `768px` |
| Feature Unit title | one table line plus tooltip, two card lines |
| timestamps | relative visible text, absolute RFC 3339 on hover/focus |
| attention reasons | highest reason visible; remaining count shown as text |

Korean prose uses `word-break: keep-all`. Machine identifiers use monospace.
Color is never the only state signal.

## 9. State Matrix

| State | Rendering | Focus/live behavior |
|---|---|---|
| `initial_loading` | eight skeleton rows matching final row heights | no live announcement |
| `ready` with results | list, summary, snapshot time | normal navigation |
| `ready` with no Projects | neutral empty state; no fake creation CTA | focus remains on heading |
| `filtered_empty` | filter controls plus query summary and `필터 초기화` | reset button reachable |
| `refreshing` | preserve rows; nonblocking progress near snapshot time | no focus steal |
| `loading_more` | append four skeleton rows after current rows | preserve current focus |
| `error_without_data` | request ID, retry, clear explanation | focus error summary if user-triggered |
| `error_with_stale_data` | preserve stale data and show retry | announce stale state politely |
| `disconnected` | visible stale connection state and reconnect progress | no row mutation |
| `denied` | non-disclosing access explanation | link back to permitted area |

Background refresh failure does not steal focus. User-triggered retry failure
may focus the error summary.

## 10. Real-Time Behavior

The route opens `GET /v1/events` only after the first REST snapshot succeeds.
It sends `Last-Event-ID` after reconnect when available.

Allowed list-affecting event summary:

```text
project.attention.changed
  projectKey
  operationalStatus
  attentionSeverity
  attentionReasons
  resourceVersion
```

Event handling:

1. If the event may affect current filters, increment `pendingUpdateCount`.
2. Do not reorder rows or mutate row details from SSE.
3. Show `새 업데이트 N건`.
4. On `업데이트 적용`, abort in-flight list request and refetch first page.
5. On disconnect, keep last data, mark stale, reconnect with bounded backoff,
   then refetch first page before clearing stale state.

Critical incident events show a persistent visible alert and concise live
announcement. They do not expose raw incident content.

## 11. Accessibility

- The route has one `<main>` landmark and one `h1`.
- The table uses native `<table>`, `<caption>`, `<th scope="col">`, and
  `aria-sort` only on the active sorted header.
- Sort controls are buttons inside header cells.
- Cards preserve visible labels for values.
- The mobile filter dialog follows modal dialog behavior: focus trap, Escape
  close, visible close, apply/reset actions, and focus return.
- Noncritical update counts use one visually hidden `role="status"`
  `aria-live="polite"` `aria-atomic="true"` node.
- Critical incidents use a persistent alert region.
- Skeletons do not announce as live content.
- Reduced motion disables nonessential transitions.

## 12. Mock Fixtures

Required fixtures before UI implementation:

| Fixture | Purpose |
|---|---|
| `projects-empty.json` | no registered Projects |
| `projects-filtered-empty.json` | query/filter with no result |
| `projects-attention-mixed.json` | healthy, action-required, blocked, incident hold, archived |
| `projects-long-content.json` | long names, keys, descriptions, Feature Unit titles |
| `projects-stale-disconnected.json` | stale snapshot with disconnected SSE |
| `projects-error-with-stale-data.json` | refresh failure preserving previous data |
| `projects-denied.json` | non-disclosing authorization state |

Fixtures must contain no secrets, raw repository credentials, raw logs, raw
provider output, or production data.

## 13. Playwright Scenarios

Required scenarios:

1. first load renders heading, filters, summary, and Project list;
2. invalid query from API renders error with request ID;
3. search debounce cancels older response and uses `router.replace`;
4. attention multi-filter updates URL without extra history entries;
5. sort header updates `aria-sort`;
6. mobile filter dialog traps focus, applies, resets, closes with Escape, and
   returns focus;
7. `Load more` appends items without losing focus;
8. SSE event increments pending update count without reordering rows;
9. applying updates refetches first page and clears pending count;
10. disconnect marks stale state and reconnect refetches before clearing stale;
11. long Project key has ellipsis, tooltip, and copy action;
12. table/card switch preserves labels and links;
13. denied state reveals no Project names;
14. reduced motion removes nonessential transitions.

Viewport evidence:

```text
320x568
390x844
768x1024
1024x768
1280x900
1440x900
1920x1080
```

Required visual states:

```text
ready
filtered_empty
stale_disconnected
error_without_data
error_with_stale_data
long_content
mobile_filter_dialog
```

## 14. Acceptance Evidence

P-01 implementation is acceptable only when evidence includes:

1. OpenAPI contract for `GET /v1/projects`;
2. controller tests for query validation and response shape;
3. application/read-model tests for projection precedence and bounded query
   behavior;
4. SSE tests for post-commit event behavior and reconnect refetch;
5. UI tests for URL state, debounce, mobile filters, keyboard paths,
   `aria-sort`, and no automatic row reorder;
6. visual captures at the viewport matrix;
7. accessibility scan and keyboard scenario results;
8. human verification note confirming the owner can identify and open the most
   urgent Project without a destructive global action.

## 15. Explicit Non-Implementation

P-01 must not implement:

- Project creation;
- Project edit/settings form;
- pause, retry, cancel, merge, deploy, or Worker execution;
- direct state mutation;
- raw log/artifact/provider rendering;
- row-as-button behavior;
- infinite scroll;
- synthetic percentage progress;
- hidden global counts outside the authorized filtered result.

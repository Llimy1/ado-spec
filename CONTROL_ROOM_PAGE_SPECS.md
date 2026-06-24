# ADO Control Room Page Contracts

This document converts the Control Room architecture and design system into
implementation-level page contracts. It is normative for `apps/control`,
`apps/api`, `packages/contracts`, and the read-model queries serving the
Control Room.

This document does not permit a browser to derive orchestration truth, change
business state directly, or receive raw runner/provider content. The database
and the typed REST API remain authoritative as defined in
`CONTROL_ROOM_API_UI_SPEC.md`.

## 1. Contract Format

Every Control Room page specification must define:

1. route, permitted actors, primary question, and explicit non-goals;
2. REST and SSE dependencies, DTOs, pagination, cache behavior, and error
   semantics;
3. database read-model ownership and computed-value provenance;
4. implementation boundaries between route, feature module, shared component,
   and generated API client;
5. exact interaction, keyboard, focus, live-region, and dialog behavior;
6. desktop, compact desktop, tablet, and mobile layout behavior;
7. loading, empty, filtered-empty, stale, disconnected, error, and denied
   states; and
8. controller, application, integration, UI, visual, and human-verification
   evidence.

`P-01` below is the first complete page contract. Later pages use the same
format and may not weaken its shared rules without an approved design-system
revision.

---

## P-01: Projects

### P-01.1 Identity And Boundaries

| Property | Contract |
|---|---|
| Route | `/projects` |
| Permitted actor | authenticated `Human Owner` in v1 |
| Primary question | Which Project needs human attention now, and where should the owner open it? |
| Read authority | `GET /v1/projects`; global SSE is freshness-only |
| State-changing commands | none |
| Non-goals | Project creation, Project configuration editing, pause/retry/cancel, direct status mutation, raw logs, and arbitrary cross-Project bulk actions |

The page is the global operational index, not a marketing dashboard or a
generic data-admin table. It presents one row per Project and routes the owner
to the relevant Project, Feature Unit, or Decision detail. It does not expose
controls that lack the context required for a safe policy decision.

### P-01.2 API And Read Model Contract

`GET /v1/projects` is expanded into the following list endpoint. Existing
listing semantics remain compatible; the query and DTO below are the required
v1 Control Room shape.

```text
GET /v1/projects
  ?q={optional 1..120 character query}
  &archived={true|false}             (optional; default false)
  &attention={action_required|blocked|incident_hold} (repeatable, optional)
  &sort={attention|activity|name}    (default attention)
  &direction={asc|desc}              (default desc except name asc)
  &cursor={opaque keyset cursor}
  &limit={1..100, default 25}
```

Unknown parameters, invalid enum values, malformed cursors, and limits above
100 return `400` with a stable machine error code. `q` is trimmed before
validation; blank `q` is omitted. Repeated filters use OR inside a filter group
and AND across filter groups. All sorting uses a deterministic `projectKey`
tie-breaker.

The endpoint returns `Cache-Control: private, no-store`. The Next.js route
fetches its first snapshot with `cache: 'no-store'`; browser navigation never
reuses a persisted sensitive control-plane list as authoritative state.

```ts
type ProjectOperationalStatus =
  | 'healthy'
  | 'attention_required'
  | 'blocked'
  | 'incident_hold'
  | 'archived'

type ProjectAttentionReason =
  | 'human_decision_required'
  | 'blocked_feature_unit'
  | 'blocked_component_work'
  | 'verification_failed'
  | 'accepted_review_finding'
  | 'open_incident'

interface ProjectListItem {
  projectKey: string
  name: string
  description: string | null
  lifecycle: 'active' | 'archived'
  operationalStatus: ProjectOperationalStatus
  attention: {
    severity: 'none' | 'warning' | 'critical'
    reasons: ProjectAttentionReason[]
    humanDecisionCount: number
    blockedCount: number
    failedCount: number
    incidentCount: number
  }
  activeFeatureUnit: {
    featureUnitKey: string
    title: string
    state: string
  } | null
  componentWork: {
    activeCount: number
    openCount: number
  }
  lastCommittedEventAt: string // RFC 3339 UTC
  resourceVersion: string
  links: {
    self: string
    overview: string
    activeFeatureUnit?: string
  }
}

interface ProjectListResponse {
  items: ProjectListItem[]
  page: {
    limit: number
    nextCursor: string | null
    hasMore: boolean
  }
  summary: {
    scope: 'filtered_result'
    totalCount: number
    actionRequiredCount: number
    blockedCount: number
    incidentHoldCount: number
  }
  snapshot: {
    observedAt: string // RFC 3339 UTC
    requestId: string
  }
}
```

`operationalStatus` is a read-model projection, never a persisted substitute
for `StateSubject` and never a writable project status. `lifecycle` maps only
to `projects.Project.is_archived`; it is not a Project state machine. The
operational-status precedence is:

```text
archived
> incident_hold
> blocked
> attention_required
> healthy
```

For active Projects, `incident_hold` applies when a relevant open incident or
held subject prevents automation. `blocked` applies when a required active
Feature Unit or Component Work is blocked. `attention_required` applies when
there is a pending human decision, failed verification, or accepted revision
finding without a higher-precedence condition. `healthy` means none of those
conditions are present; it is not a claim that every historical run succeeded.

The API owns this projection. Its query is one bounded `ProjectListProjection`
read query using joins/subqueries or a materialized read model; it must not
load Project rows and then issue one ORM relation query per row. The query
selects only redacted repository-independent metadata needed by this page.

Cursor pagination is keyset pagination, not offset pagination. A cursor encodes
the selected sort version, last sort tuple, and `projectKey`, and is signed or
otherwise tamper-evident. It is intentionally *not* a durable cross-request
database snapshot: a Project may change between pages. The response exposes
`observedAt`; the client de-duplicates `projectKey` while appending pages and
offers a fresh first-page refetch after a relevant SSE event. This avoids
pretending the list has snapshot isolation it cannot prove.

### P-01.3 URL And Client State

The URL is the only durable browser state for filters and sorting.

```text
/projects?q=orion&attention=blocked&attention=action_required&sort=attention&direction=desc
```

The default URL omits default values. The search input applies after a `250ms`
debounce with `router.replace`; selecting a filter, clearing filters, and
changing sort also use `router.replace`. Typing must not create one history
entry per keystroke. The selected Project is represented by normal navigation,
not by a local selected-row state.

The client state machine has exactly these display phases:

```text
initial_loading
ready
refreshing
loading_more
error_without_data
error_with_stale_data
disconnected
```

It stores `itemsByProjectKey`, rendered order, `nextCursor`, current query,
summary, `snapshot.observedAt`, `pendingUpdateCount`, and the newest pending
event severity. A request sequence number and `AbortController` prevent an
older delayed search response from replacing a newer result. "Load more" is a
button, not automatic infinite scroll, so focus and footer controls remain
predictable.

### P-01.4 Real-Time Algorithm

The page opens `GET /v1/events` only after the first REST snapshot succeeds.
It sends `Last-Event-ID` after a reconnect when one exists. Global event
payloads have the existing safe event envelope and may only use the global
event types defined in `CONTROL_ROOM_API_UI_SPEC.md`. A
`project.attention.changed` event summary contains only `operationalStatus`,
`attentionSeverity`, `attentionReasons`, and `resourceVersion`, allowing the
client to decide whether the Project could match its current filter without
fetching protected detail data.

```text
on global event
  -> if the event affects a Project that could match current filters:
       increment pendingUpdateCount
       retain current table order and row data
  -> if event is incident.updated or severity is critical:
       show persistent critical banner and announce one concise message
  -> otherwise announce only "새 업데이트 N건" through role=status

on "업데이트 적용"
  -> abort in-flight list request
  -> fetch first page with the current URL query
  -> replace list, summary, observedAt, and cursor atomically
  -> clear pendingUpdateCount

on stream disconnect
  -> retain last REST data, mark it stale, show reconnect state
  -> reconnect with bounded backoff
  -> after reconnect, fetch the first page before removing stale state
```

No event silently reorders rows, changes a selected/focused row, or asserts
command success. The page uses one visually hidden `role="status"`
`aria-live="polite"` `aria-atomic="true"` node for noncritical update counts.
Critical incidents use a persistent visible alert region and do not expose raw
incident data in the stream.

### P-01.5 Desktop Information Architecture

At `1280px` and above, the page renders in this order:

```text
breadcrumb
page title + result count + last authoritative snapshot time
search + filter controls + sort control + connection freshness
filtered-result attention summary
semantic Project table
load-more / end-of-results region
```

The summary is a compact, non-clickable readout: `사람 판단 필요`, `차단됨`,
and `사고 보류`. It describes the currently filtered result, not an undisclosed
global count. A summary cell links nowhere and has no implied command.

The full table uses native HTML table semantics:

| Column | Width rule | Content |
|---|---|---|
| Project | `minmax(240px, 2fr)` | named link, `projectKey`, optional one-line description |
| Operational status | `132px` | icon, localized status text, reason count |
| Active Feature Unit | `minmax(220px, 1.5fr)` | direct Feature Unit link or `활성 작업 없음` |
| Component Work | `128px` | active/open counts, descriptive only |
| Attention | `160px` | highest reason plus remaining count |
| Last committed activity | `156px` | relative time and focusable absolute-time tooltip |
| Open | `48px` | labeled link with visible icon |

The table has a visually hidden `<caption>` identifying the current result and
filter state. Header cells use `<th scope="col">`; sortable headers contain
real `<button>` controls and the sorted `<th>` carries `aria-sort`. Project and
Feature Unit values are links. Rows are not click targets and do not use
`role="button"`; this keeps normal Tab order and avoids an invalid nested-link
or fake-row interaction. Since this is not an editable cell-navigation
interface, it must not implement ARIA `grid`.

### P-01.6 Compact, Tablet, And Mobile Layout

| Range | Navigation | Primary representation | Hidden or relocated information |
|---|---|---|---|
| 1280px+ | 256px sidebar | full semantic table | none |
| 1024-1279px | 256px sidebar | compact semantic table | Component Work moves into Project cell; Open column removed because Project name link remains |
| 768-1023px | 64px rail | card list, two-column metadata | Feature Unit and attention remain; description and Component Work secondary counts move to card detail |
| 320-767px | Drawer | one-column card list | one highest reason, active Feature Unit, and activity shown; remaining metadata opens in Project Overview |

The mobile filter button opens a true modal dialog. It contains one labelled
form, `적용` and `초기화` controls, a visible close control, focus containment,
`Escape` close, and focus return to the invoking filter button. Choosing a
filter does not apply it until `적용` is pressed. The desktop filter controls
are inline and apply immediately through the URL.

Every text policy is explicit:

- Project name: maximum two lines on card layouts; one line in table layouts.
- `projectKey`: one monospace line with ellipsis, copy button, and tooltip.
- Description: one line on desktop, hidden below `768px`.
- Feature Unit title: two lines on card layouts; one table line plus tooltip.
- Timestamps: relative visible text; absolute RFC 3339 equivalent on focus and
  hover, never color-only freshness.
- The page itself never horizontally scrolls. Only the full table wrapper may
  horizontally scroll between `1024px` and `1279px` when user font scaling or
  localization makes its compact column minimum impossible; the wrapper has a
  visible scroll affordance and keyboard-scrollable overflow.

### P-01.7 State Matrix

| State | Required rendering | Allowed action |
|---|---|---|
| initial loading | eight skeleton rows matching final row heights; controls remain disabled | none |
| ready, results | list plus current snapshot time | search, filter, sort, navigation, load more |
| ready, no Projects | neutral empty state explaining that no Project is registered | no fake creation action |
| ready, filtered empty | retain filter controls, show query summary and `필터 초기화` | clear filters |
| refreshing | preserve rows and their geometry; show nonblocking progress near snapshot time | cancel only by changing query/navigation |
| loading more | append four skeleton rows after existing rows; preserve current focus | none |
| error without data | error summary with request ID and retry | retry |
| error with stale data | preserve last data, mark stale time, show retry | retry |
| disconnected | visible stale connection state, last successful snapshot, reconnection progress | manual refresh |
| denied | route-level authorized explanation without revealing undiscoverable Project names | navigate to permitted area |

No skeleton announces as live content. Errors use a focused summary only when
the user-triggered action failed; background refresh failures do not steal
focus. A visually present retry action is always keyboard accessible.

### P-01.8 Planned Frontend Boundaries

The following are target file boundaries for implementation, not currently
existing files:

```text
apps/control/app/(control)/projects/page.tsx
apps/control/features/projects/project-list-route.tsx
apps/control/features/projects/project-list-table.tsx
apps/control/features/projects/project-list-cards.tsx
apps/control/features/projects/project-filter-controls.tsx
apps/control/features/projects/project-attention-summary.tsx
apps/control/features/projects/project-list-state.ts
apps/control/features/events/use-global-control-events.ts
packages/contracts/src/projects/project-list.contract.ts
packages/contracts/src/events/global-control-event.contract.ts
```

`page.tsx` obtains the initial no-store snapshot through the generated client.
The feature route owns query synchronization and the page reducer. Presentational
table/card components receive typed view models and callbacks only; they do not
open EventSource connections, construct URLs, inspect HTTP errors, or derive
operational status. The generated OpenAPI client owns transport serialization.

### P-01.9 Verification Contract

Required evidence before a Project-list implementation can be accepted:

1. API controller tests reject invalid filters, cursor, and limit; generated
   OpenAPI exposes the full query and response schema.
2. Application/integration tests prove projection precedence for archived,
   incident hold, blocked, attention required, and healthy cases, and prove no
   N+1 relation query pattern for a 25-item page.
3. SSE integration tests prove global events publish only after transaction
   commit, omit raw content, and force a REST refetch after reconnect.
4. UI tests prove URL state, debounce cancellation, mobile apply/reset,
   keyboard navigation, `aria-sort`, live update announcement, and no automatic
   row reorder.
5. Visual tests cover `320x568`, `390x844`, `768x1024`, `1024x768`,
   `1280x900`, `1440x900`, and `1920x1080` for ready, filtered-empty, stale,
   error, and long-content cases.
6. Human verification confirms that an owner can identify the most urgent
   Project without opening every row and can reach that Project without an
   ambiguous or destructive global action.

### P-01.10 Standards References

- [WAI-ARIA Grid Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/grid/): use
  native tables unless composite-cell navigation is genuinely required.
- [WAI-ARIA Modal Dialog Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/): modal focus, escape, and focus-return behavior.
- [WCAG 2.2 Reflow](https://www.w3.org/WAI/WCAG22/Understanding/reflow.html):
  responsive reflow requirements.
- [WCAG 2.2 Focus Not Obscured](https://www.w3.org/WAI/WCAG22/Understanding/focus-not-obscured-minimum.html): focus must remain perceivable.
- [W3C ARIA status technique](https://www.w3.org/WAI/WCAG20/Techniques/aria/ARIA22): polite live announcement for result/update status.
- [MDN Container Queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_container_queries): component-local layout adaptation.

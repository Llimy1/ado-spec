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

---

## P-02: Project Overview

### P-02.1 Identity And Boundaries

| Property | Contract |
|---|---|
| Route | `/projects/{projectKey}` |
| Permitted actor | authenticated `Human Owner` authorized for the Project |
| Primary question | Is this Project advancing safely, what must the owner decide, and which exact work record proves that answer? |
| Read authority | `GET /v1/projects/{projectKey}/overview`; Project SSE is freshness-only |
| State-changing commands | none on the overview itself |
| Non-goals | a generic KPI dashboard, direct pause/retry/cancel, Project configuration editing, raw logs, mutable checklist items, and a visual graph as the only source of dependency truth |

The overview is a project-scoped decision surface. It is deliberately not a
collection of decorative metrics. Every number, status, and alert links to the
Feature Unit, Component Work, decision, incident, or event record that proves
it. Operational actions belong to their specific detail page, where policy,
evidence, version, and consequence can be shown together.

### P-02.2 Overview Read Model

The existing endpoint has this required response shape:

```text
GET /v1/projects/{projectKey}/overview
```

It accepts no free-text filter and returns one bounded, purpose-built
`ProjectOverviewProjection`. It is not assembled in the browser from unrelated
list endpoints. It returns `Cache-Control: private, no-store` and includes a
server `observedAt` timestamp, `requestId`, and `resourceVersion`.

```ts
type AttentionSeverity = 'none' | 'warning' | 'critical'

interface ProjectOverviewResponse {
  project: {
    projectKey: string
    name: string
    description: string | null
    archived: boolean
    timezone: string
    repository: {
      repositoryKey: string
      remoteUrlRedacted: string
      integrationBranch: string
      defaultBranch: string
    } | null
    activeConstraintProfile: {
      version: number
      approvedAt: string
    } | null
  }
  attention: {
    severity: AttentionSeverity
    operationalStatus:
      | 'healthy'
      | 'attention_required'
      | 'blocked'
      | 'incident_hold'
      | 'archived'
    reasons: Array<{
      code: ProjectAttentionReason
      count: number
      href: string
    }>
    nextRequiredHumanAction: {
      decisionKey: string
      title: string
      dueAt: string | null
      href: string
    } | null
  }
  roadmap: {
    roadmapKey: string
    title: string
    state: string
    requiredFeatureUnit: {
      total: number
      closed: number
      active: number
      blocked: number
      awaitingHuman: number
    }
    href: string
  } | null
  focusFeatureUnit: {
    featureUnitKey: string
    title: string
    state: string
    reason: 'requires_human_action' | 'blocked' | 'active' | 'next_approved'
    componentWork: {
      total: number
      running: number
      blocked: number
      needsRevision: number
      readyForPr: number
      prCreated: number
    }
    href: string
  } | null
  componentWork: {
    items: Array<{
      componentWorkKey: string
      title: string
      primaryComponent: { key: string; displayName: string }
      executionScope: 'single' | 'coordinated'
      state: string
      attentionSeverity: AttentionSeverity
      activeRun: { jobAttemptId: string; startedAt: string; href: string } | null
      lastCommittedEventAt: string
      href: string
    }>
    totalOpenCount: number
    omittedCount: number
  }
  executionFocus: {
    activeAttemptCount: number
    primary: {
      jobKey: string
      jobAttemptId: string
      agentRunId: string | null
      componentWorkKey: string
      phase: string
      runnerLabel: string
      startedAt: string
      timeoutAt: string
      href: string
    } | null
    capacity: {
      eligibleReadyWorkerCount: number
      eligibleDegradedWorkerCount: number
      queueDepthForProject: number
      oldestQueuedAt: string | null
    }
  }
  decisionQueue: {
    items: Array<{
      decisionKey: string
      title: string
      targetType: 'feature_unit' | 'component_work' | 'incident' | 'configuration'
      requestedAt: string
      dueAt: string | null
      severity: AttentionSeverity
      href: string
    }>
    totalPendingCount: number
    omittedCount: number
  }
  recentActivity: {
    items: Array<{
      eventKey: string
      occurredAt: string
      actorLabel: string
      kind: 'state_transition' | 'verification' | 'review' | 'decision' | 'incident' | 'pull_request'
      summary: string
      subject: { type: string; key: string; href: string }
      evidenceHref: string | null
    }>
    omittedCount: number
  }
  dependencyMap: {
    isTruncated: boolean
    omittedNodeCount: number
    nodes: Array<{
      featureUnitKey: string
      title: string
      state: string
      attentionSeverity: AttentionSeverity
      href: string
    }>
    edges: Array<{
      fromFeatureUnitKey: string
      toFeatureUnitKey: string
      relation: 'depends_on'
    }>
    accessibleRows: Array<{
      featureUnitKey: string
      dependsOn: string[]
      blocks: string[]
    }>
  }
  snapshot: {
    observedAt: string
    resourceVersion: string
    requestId: string
  }
}
```

The endpoint returns at most 12 Component Work items, 3 pending decisions, 10
recent activity entries, 24 dependency nodes, and 48 dependency edges. It
sets `omittedCount` or `isTruncated` rather than silently hiding data. Deeper
lists belong to dedicated detail routes. No raw artifact payload, command
argument, local path, provider token, or unredacted runner log is included.

### P-02.3 Deterministic Projection Rules

The API computes each overview section in the application/read-model layer.
The Control Room must not recreate these rules from child records.

| Projection | Rule |
|---|---|
| Project operational status | the P-01 precedence contract, scoped to this Project |
| Focus Feature Unit | first required Unit with pending human action; else first blocked required Unit; else oldest active Unit by `sequence_number`; else first approved Unit by `sequence_number`; else `null` |
| Component Work items | open Work sorted by attention severity, then active-run presence, then latest committed event descending, then `componentWorkKey` |
| Primary execution | oldest currently running JobAttempt; ties break by Job priority then attempt key |
| Worker capacity | Workers eligible for this Project's queued/running job capabilities; it is not a global worker-pool health claim |
| Recent activity | append-only AuditEvent/StateTransition projection ordered by `occurredAt desc`, then immutable event key |
| Dependency graph | only unwaived required `depends_on` FeatureUnitRelation edges; `blocks` is presented in textual rows, not drawn as a second ambiguous arrow type |

The overview query uses a bounded collection of SQL projections or materialized
read models, executed under a single read service. A controller cannot invoke
one repository per panel. The implementation records and tests a query budget
for this endpoint; adding a panel must not introduce N+1 child reads.

`focusFeatureUnit.reason` is rendered in Korean as an explanation, for example
`사람의 승인 대기`, `차단된 작업`, `현재 실행 중`, or `다음 승인 단위`. It is
not a mutable state label.

### P-02.4 Page Layout And Information Priority

At `1440px` and above, use the 12-column shell grid in this exact order:

```text
breadcrumb
Project identity header and freshness
attention band, only when severity is warning or critical
row 1: focus Feature Unit (8 columns) | pending human decisions (4 columns)
row 2: Component Work table (8 columns) | execution focus and capacity (4 columns)
row 3: recent activity timeline (7 columns) | dependency map (5 columns)
```

The header shows Project name, archived marker when applicable, `projectKey`,
redacted repository URL, integration branch, active constraint-profile version,
and the last authoritative snapshot. Branch values are descriptive; no branch
control is present. Long machine values follow the global copy-and-tooltip
policy.

The attention band is absent for `healthy`. For warning and critical states it
appears before all summaries and contains: severity label, the highest-priority
reason, count, a link to the proving record, and one next required human
action when present. It is not dismissible until the underlying REST projection
changes.

The page never displays synthetic velocity, MTTR, success percentage, change
failure rate, model quality score, or predicted completion time in v1. Those
metrics require separately defined calculation windows and evidence rules; a
plausible-looking number is less safe than an explicit absence.

### P-02.5 Section Contracts

#### Focus Feature Unit

The focus card has a named link, current lifecycle state, selection reason,
Component Work count summary, and direct link to the Feature Unit detail. It
does not show a percent-complete bar because Feature Unit completion is a
state/evidence-gated process, not a reliable arithmetic percentage. When no
Feature Unit is selectable, render one of these API-provided reasons:

```text
no_approved_roadmap
no_feature_units
all_required_feature_units_closed
project_archived
```

The card presents the matching roadmap/detail link only. It never offers a
button to activate work directly.

#### Pending Human Decisions

This panel shows at most three decision rows. Each row contains title, target
type, requested time, optional due time, severity, and a `검토하기` link. It
contains no approve/reject controls; decisions require their dedicated context,
evidence, and explicit reason record. A `전체 N건 보기` link preserves the
Project filter in the Human Decision Inbox route.

#### Component Work Table

At wide desktop the semantic table has `Component`, `Work`, `Scope`, `State`,
`Active run`, `Last committed activity`, and `Open` columns. `Scope` exposes
`single` or `coordinated` with a localized label and a text description of
declared component roots on the detail route. State uses icon, text, and
attention severity. No row itself is clickable; the Work title and Open link
are native anchors.

The table is an overview sample, not the complete Work inventory. The footer
displays `열린 작업 전체 N건 보기` when `omittedCount > 0`, with no implicit
infinite scroll.

#### Execution Focus

The panel distinguishes exactly these facts:

```text
no active attempt
-> queued work only
-> active attempt running
-> active attempt near timeout
-> no eligible ready Worker
```

For a running attempt it renders Job/Attempt key, related Component Work,
runner label, phase, start time, timeout time, and a link to Run Detail. A
progress bar is prohibited unless the Job handler emits a defined bounded
progress model. Elapsed time is not progress. A near-timeout condition begins
when remaining time is less than 20 percent of the defined timeout and includes
the exact deadline.

#### Recent Activity

Recent Activity is a static ordered list (`<ol>`), not a live `feed` and not a
scrolling terminal. Each item contains a `<time>`, actor, concise event
summary, subject link, and optional evidence link. The list's accessible text
states the event type in addition to its color/icon. New SSE events do not
prepend into the list while it is being read; they increment the page-level
pending-update indicator.

#### Dependency Map

The map is a supplementary visualization of required `depends_on` edges, not
an interactive `tree` or a source of action. It renders no clickable SVG nodes
and has no custom arrow-key behavior. This avoids falsely declaring tree/grid
semantics for a directed acyclic relation that users do not edit here.

The implementation uses a bounded, deterministic layout: assign each node a
layer equal to one plus the maximum layer of its prerequisites, stable-sort
same-layer nodes by `sequence_number` then key, render SVG arrows behind
noninteractive nodes, and recompute connector geometry using `ResizeObserver`.
The visible figure has a short caption. Immediately adjacent to it is a native
table/list representation built from `accessibleRows`; it is visible by default
to assistive technology and available to sighted users through `의존성 목록
보기`. At less than `768px`, the SVG is not rendered and the textual list is
the sole representation. A truncated map states its omitted-node count and
links to the Roadmap/Feature Unit view.

### P-02.6 Real-Time And Staleness Behavior

After the REST snapshot succeeds, the page opens
`GET /v1/projects/{projectKey}/events`. It keeps a local `pendingUpdateCount`
and `highestPendingSeverity`; the current overview stays spatially stable until
the owner chooses `업데이트 적용`.

```text
state.transitioned / verification.completed / review.finding.created
  -> increment pending count only

human_decision.required / incident.updated
  -> increment pending count; show persistent attention banner
  -> announce concise non-sensitive status through the page live region

job.attempt.updated
  -> update only the explicit connection freshness indicator; do not animate
     elapsed/progress or replace the execution card

stream reconnect
  -> mark all summary data stale
  -> re-fetch overview REST snapshot
  -> atomically replace all panels and clear pending count
```

If a Project becomes archived, the next authoritative snapshot changes the
header and makes all overview links read-only navigation. If access is revoked,
the next REST response returns the existing undiscoverable `404` behavior; the
page clears in-memory Project data before rendering the permission state.

### P-02.7 Responsive Contract

| Range | Layout | Required adaptation |
|---|---|---|
| 1440px+ | 12-column wide layout | all sections shown as in P-02.4 |
| 1280-1439px | 8-column desktop layout | focus/decisions and work/execution remain two columns; timeline/map become stacked if either panel reaches its min width |
| 1024-1279px | 8-column content with sidebar | attention, focus, decisions, execution stack; Component Work becomes compact table; dependency map moves below activity |
| 768-1023px | 6-column content with rail | all panels one column except compact attention counts; Component Work renders cards; only textual dependency list |
| 320-767px | 4-column content with Drawer | header metadata becomes disclosure rows; decision list, focus card, execution card, work cards, activity, and dependency list are single column |

No overview panel relies on viewport width alone for its internal reflow.
Cards with an optional side metric declare `container-type: inline-size` and
switch to stacked content when their own width falls below `420px`. The global
shell controls navigation breakpoints; components control internal density.

### P-02.8 Accessibility And Focus Rules

- The route renders one `<main>` landmark and one `h1` containing Project name.
  Client-side navigation moves focus to this heading after loading a new Project.
- Each overview panel is a labelled `<section>` with a unique heading. The
  attention band is the first focusable/announced region after the header when
  critical; otherwise normal document order remains unchanged.
- Tooltip-only machine-value expansion is supplementary. The copy button has a
  visible accessible name such as `integration branch 복사`; tooltip content is
  referenced with `aria-describedby` and never contains an action.
- The mobile filter/dialog conventions from P-01 apply to every overview
  dialog. No panel opens a modal solely to display information that could be a
  route or disclosure.
- The dependency figure follows the complex-image rule: short caption plus
  equivalent structured description. The adjacent relationship list is not
  hidden from assistive technology.
- A connection update, background snapshot, or SSE message never moves DOM
  focus. Only user-initiated navigation or a failed user command may change
  focus.

### P-02.9 State Matrix

| Condition | Required result |
|---|---|
| initial loading | header and panel skeletons preserve final geometry; no fake values or live announcement |
| healthy Project | no attention band; all fact panels render from snapshot |
| warning / critical | attention band and proving links render before summary panels |
| archived Project | archived marker, read-only context, no execution urgency claim |
| no roadmap | roadmap-empty explanation, no Feature Unit/graph fabrication |
| no focus Feature Unit | explicit API reason and roadmap link when permitted |
| no active attempt | execution panel renders queue/capacity state, not an empty chart |
| no eligible ready Worker | execution panel is warning/critical according to API severity and links to Worker context |
| dependency truncation | visible count plus complete textual route; never silently remove nodes |
| stale/disconnected | preserve snapshot, show observed time and connection state, disable only freshness-dependent affordances |
| error with snapshot | retain last snapshot with stale marker and retry; do not clear identifying header until access is denied |
| error without snapshot | route-level error with request ID and retry; no guessed Project name |
| denied/not found | clear cached data, show non-discoverable access result, provide Projects navigation |

### P-02.10 Planned Frontend Boundaries

```text
apps/control/app/(control)/projects/[projectKey]/page.tsx
apps/control/features/project-overview/project-overview-route.tsx
apps/control/features/project-overview/project-attention-band.tsx
apps/control/features/project-overview/focus-feature-unit-card.tsx
apps/control/features/project-overview/pending-decision-list.tsx
apps/control/features/project-overview/component-work-summary-table.tsx
apps/control/features/project-overview/execution-focus-panel.tsx
apps/control/features/project-overview/recent-activity-timeline.tsx
apps/control/features/project-overview/feature-dependency-figure.tsx
apps/control/features/project-overview/feature-dependency-list.tsx
apps/control/features/project-overview/project-overview-state.ts
packages/contracts/src/projects/project-overview.contract.ts
```

The server route obtains the initial generated-client response. The route
feature owns the Project SSE subscription, freshness reducer, and atomic
snapshot replacement. Panel components receive typed slices only. The SVG
dependency renderer receives already authorized nodes/edges and cannot fetch
data, calculate state, or open an EventSource.

### P-02.11 Verification Contract

1. API/OpenAPI tests validate all bounded collection limits, redaction, and
`null` cases in the Overview DTO.
2. Projection tests prove focus-Feature-Unit priority, operational-status
precedence, active-attempt selection, and dependency-edge filtering.
3. Database integration tests prove a bounded query budget for Projects with
12 Component Works, 3 decisions, 10 events, and 24 dependency nodes.
4. UI tests prove no synthetic percentages/progress, correct empty variants,
stable ordering during SSE updates, focus retention, and all proving links.
5. Accessibility tests verify semantic sections, native table/list semantics,
keyboard navigation, dependency text equivalence, tooltip behavior, and no
obscured focus at every responsive range.
6. Visual evidence covers healthy, blocked, decision-required, archived,
worker-unavailable, dependency-truncated, stale, and error states at the
global design-system viewport matrix.
7. Human verification proves that the owner can answer the primary question
without interpreting an undocumented metric or opening a raw log.

### P-02.12 Standards References

- [W3C Complex Images](https://www.w3.org/WAI/tutorials/images/complex/):
  diagram/graph requires a concise description and equivalent long structured
  representation.
- [WAI-ARIA Tree View Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/treeview/):
  do not use tree semantics without the required hierarchy and keyboard model.
- [WAI-ARIA Tooltip Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/tooltip/):
  tooltip trigger, focus, escape, and `aria-describedby` behavior.
- [MDN Container Queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_container_queries):
  component-local responsive adaptation.

---

## P-03: Roadmaps

### P-03.1 Identity And Routes

| Property | Roadmap index | Roadmap detail |
|---|---|---|
| Route | `/projects/{projectKey}/roadmaps` | `/projects/{projectKey}/roadmaps/{roadmapKey}` |
| Primary question | Which approved or pending plan governs this Project? | Is this Roadmap valid, approved, and decomposed into executable Feature Units without unresolved dependency risk? |
| Read authority | `GET /v1/projects/{projectKey}/roadmaps` | `GET /v1/projects/{projectKey}/roadmaps/{roadmapKey}` |
| State-changing actions | none | record an allowed human planning decision |
| Non-goals | document editing, markdown source overwrite, Feature Unit implementation control, direct state editing, merge/deploy control | same |

Roadmaps are human-provided planning sources made executable only after analysis
and approval. The UI never treats a copied markdown document or an Agent
summary as approved truth. It exposes source provenance, current plan state,
Feature Unit decomposition, dependency conditions, and the exact human decision
required before execution.

### P-03.2 API Contract

The index endpoint is a new bounded read projection:

```text
GET /v1/projects/{projectKey}/roadmaps
  ?state={draft|analyzed|review_ready|approved|active|completed|archived|blocked|cancelled|incident_hold}
  &sort={activity|sequence|name}    (default activity desc)
  &cursor={opaque keyset cursor}
  &limit={1..100, default 25}
```

The detail endpoint has no client-calculated state and returns the following
projection. Both endpoints are `private, no-store` and return `requestId`,
`observedAt`, and `resourceVersion`.

```ts
interface RoadmapListItem {
  roadmapKey: string
  title: string
  state: string
  source: { version: string; importedAt: string; contentSha256: string }
  featureUnitSummary: {
    total: number
    approved: number
    active: number
    blocked: number
    closed: number
  }
  pendingPlanningDecision: boolean
  lastCommittedEventAt: string
  href: string
}

interface RoadmapDetailResponse {
  roadmap: {
    roadmapKey: string
    title: string
    summary: string
    state: string
    source: {
      artifactKey: string
      sourceKind: string
      sourceVersion: string
      contentSha256: string
      importedAt: string
      renderedDocumentHref: string | null
    }
    analyzedAt: string | null
    approvedAt: string | null
    resourceVersion: string
  }
  planningGate: {
    state: 'not_ready' | 'analysis_required' | 'human_decision_required' | 'approved' | 'blocked'
    explanationCode: string
    allowedActions: Array<'record_human_decision'>
    decisionRequirements: {
      canApprove: boolean
      canRequestChanges: boolean
      changeRequestReasonRequired: boolean
      expectedResourceVersion: string
    }
  }
  featureUnits: {
    items: Array<{
      featureUnitKey: string
      sequenceNumber: number
      title: string
      state: string
      riskLevel: string
      dependency: {
        requiredPrerequisiteCount: number
        unsatisfiedPrerequisiteCount: number
        blockingDependentCount: number
      }
      componentWork: {
        requiredCount: number
        openCount: number
        blockedCount: number
        prCreatedCount: number
      }
      pendingHumanDecisionCount: number
      href: string
    }>
    totalCount: number
    omittedCount: number
  }
  dependencyMap: {
    isTruncated: boolean
    omittedNodeCount: number
    nodes: Array<{ featureUnitKey: string; title: string; state: string; href: string }>
    edges: Array<{ fromFeatureUnitKey: string; toFeatureUnitKey: string; relation: 'depends_on' }>
    accessibleRows: Array<{ featureUnitKey: string; dependsOn: string[]; blocks: string[] }>
  }
  recentPlanningActivity: {
    items: Array<{
      eventKey: string
      occurredAt: string
      actorLabel: string
      summary: string
      evidenceHref: string | null
    }>
    omittedCount: number
  }
  snapshot: { observedAt: string; requestId: string; resourceVersion: string }
}
```

The detail response returns at most 50 Feature Units, 24 graph nodes, 48 graph
edges, and 10 planning events. A larger Roadmap remains navigable: the Feature
Unit list stays cursor-paginated through its dedicated route, and the response
reports omission instead of truncating silently.

### P-03.3 Planning Decision And Command Contract

`analyzed -> review_ready` is a system transition gated by a valid
`RoadmapAnalysis`; the browser has no command for it. Roadmap planning approval
requires a HumanDecision, so the Control API exposes only the following human
command. It creates a typed TransitionRequest and never writes a Roadmap state
directly.

```text
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/commands/record-human-decision
```

Both require `Idempotency-Key`. The record-decision request body is:

```ts
interface RecordRoadmapPlanningDecisionRequest {
  decision: 'approved' | 'changes_requested'
  expectedResourceVersion: string
  reason?: string
}
```

`changes_requested` requires a non-empty, trimmed reason of 10 to 2,000
characters. `approved` may include an optional 2,000-character rationale.
The API validates the current allowed action, authenticated actor, expected
version, policy decision, and evidence gate. It returns `202` with a durable
TransitionRequest/HumanDecision reference when asynchronous work remains, or
`422` with a stable policy/evidence reason when the transition is not allowed.

The UI derives buttons only from `planningGate.allowedActions`. It does not
infer that `review_ready` means approval is always permitted. The approve and
request-changes controls live inside a review panel with source and decomposition
evidence links, never in the Roadmap index table.

`승인` opens an `alertdialog` because it allows downstream Feature Units to
become executable. Its initial focus is the least destructive visible action,
`취소`. `변경 요청` opens a labelled modal dialog containing a required reason
textarea. Closing either dialog returns focus to its invoker; a `409` version
conflict preserves entered text and asks the owner to refresh against the new
Roadmap version before resubmitting.

### P-03.4 Roadmap Detail Layout

At `1440px` and above, the page order is:

```text
breadcrumb
Roadmap identity, current state, source version/hash, freshness
planning-gate panel (when not approved/active/completed)
Feature Unit decomposition table
recent planning activity (7 columns) | dependency figure and relation list (5 columns)
```

The identity header makes provenance inspectable: source kind, imported time,
version, content SHA, analyzed time, and approved time. The document link opens
only a redacted, authorized render of the source Artifact. A source file is not
editable in the browser and the hash is copied through the standard machine
value control.

The planning-gate panel is the semantic center of the page. It shows current
Roadmap state, what condition is missing, the transition that becomes possible,
and the source/decomposition evidence links. It does not simply say
"approval required". For example:

```text
Current: review_ready
Required: Human planning approval
Evidence: Roadmap analysis v3; 8 Feature Units; 0 unresolved blocking cycles
Effect of approval: approved -> active is possible only after a valid approved Feature Unit exists
```

This wording distinguishes the approval decision from the later `approved ->
active` state-machine gate and prevents an operator from assuming one click
starts all work.

### P-03.5 Feature Unit Decomposition Table

At `1280px` and above, use a native table with this column contract:

| Column | Content |
|---|---|
| Sequence | immutable `sequenceNumber`, not a draggable priority editor |
| Feature Unit | title link and monospace key |
| State | icon + localized text + evidence-backed reason when exceptional |
| Risk | textual risk level, never color alone |
| Dependencies | satisfied/required count and link to relation list |
| Component Work | required/open/blocked/PR-created factual counts |
| Human gate | pending count or `없음` |
| Open | explicit detail link |

The table is read-only. It does not use ARIA grid, in-cell editing, drag and
drop, or bulk selection. Sort order is fixed to `sequenceNumber`; an operator
may filter by state/risk/dependency condition, but cannot reorder the roadmap
through the UI. Feature Unit state transitions happen only through their named
commands and StateMachine.

At `1024px` through `1279px`, Component Work counts collapse into a disclosed
text summary under the Unit title. At less than `1024px`, render semantic cards
instead of visually squeezing a table: sequence/state/risk on the first line,
title link, dependency condition, then component/human-gate facts. Desktop and
mobile render one semantic representation at a time; hidden table content is
not duplicated for screen readers.

### P-03.6 Dependency And Activity Rules

The Roadmap dependency figure follows P-02's noninteractive SVG-plus-text
contract. It depicts only `depends_on` edges. It has a short figure caption and
an adjacent semantic relationship table built from `accessibleRows`; it never
uses `tree`, `treegrid`, or a custom keyboard model. The relation table is
always available, and is the only dependency representation below `768px`.

Recent Planning Activity is a ten-item ordered list of immutable analysis,
approval, relation, and Feature Unit planning events. It is not a general log.
Each event links to its authorized evidence or subject where available. A
missing evidence link is displayed as `근거 없음` only when the event type has
no required Artifact; it must not fabricate an evidence reference.

### P-03.7 Real-Time, State, And Failure Behavior

The detail page subscribes to the Project stream after its first snapshot. It
marks updates pending without moving rows or replacing the planning-gate panel
under focus. The owner applies updates through a single `업데이트 적용` control,
which refetches the complete detail projection atomically.

| Condition | Required rendering |
|---|---|
| `draft` | source imported; analysis action/state explanation; no approval command |
| `analyzed` | analysis evidence visible; state-machine explanation for review readiness |
| `review_ready` | planning-gate panel and allowed human decision controls |
| `approved` | approval record; explain that activation still needs approved Feature Unit evidence |
| `active` | decomposition table and current execution links; no planning-edit controls |
| `completed` / `archived` | read-only history/provenance; no approval action |
| `blocked` / `incident_hold` | exceptional-state banner, proving event, and recovery/incident link; no local bypass |
| source Artifact unavailable | show metadata and redaction/unavailability reason; do not display an empty document viewer |
| relation truncation | omitted count and Feature Unit-detail route; never draw a partial graph as complete |
| conflict after decision submit | preserve typed reason, display server reason/version, require refresh before retry |
| stale/disconnected | retain snapshot with timestamp, disable only decision submission until REST refresh proves current version |
| denied/not found | clear cached roadmap/source metadata and render non-discoverable result |

### P-03.8 Planned Frontend Boundaries

```text
apps/control/app/(control)/projects/[projectKey]/roadmaps/page.tsx
apps/control/app/(control)/projects/[projectKey]/roadmaps/[roadmapKey]/page.tsx
apps/control/features/roadmaps/roadmap-list-route.tsx
apps/control/features/roadmaps/roadmap-detail-route.tsx
apps/control/features/roadmaps/roadmap-planning-gate.tsx
apps/control/features/roadmaps/record-planning-decision-dialog.tsx
apps/control/features/roadmaps/feature-unit-decomposition-table.tsx
apps/control/features/roadmaps/feature-unit-decomposition-cards.tsx
apps/control/features/roadmaps/roadmap-dependency-figure.tsx
apps/control/features/roadmaps/roadmap-relation-table.tsx
apps/control/features/roadmaps/recent-planning-activity.tsx
packages/contracts/src/roadmaps/roadmap-list.contract.ts
packages/contracts/src/roadmaps/roadmap-detail.contract.ts
packages/contracts/src/roadmaps/roadmap-commands.contract.ts
```

The generated API client owns request serialization and typed error decoding.
The decision dialog owns local form state only; it receives allowed actions and
expected version from the server projection, passes an idempotency key with the
command, and never decides eligibility itself.

### P-03.9 Verification Contract

1. OpenAPI/controller tests cover list cursor/filter validation, all detail
`null` branches, command idempotency, reason requirements, and stale-version
`409` responses.
2. Application tests prove the Roadmap state-machine gates: source validity,
analysis, human approval, and approved Feature Unit requirement for activation.
3. Integration tests prove source provenance/hash redaction, dependency-cycle
rejection, relation filtering, and bounded projection query count.
4. UI tests prove table/card semantic switching, fixed sequence order, dialog
focus/reason retention, `aria-live` update notice, and no state inference.
5. Accessibility and visual checks cover source unavailable, review ready,
approved but inactive, blocked, incident hold, 50-Unit truncation, and all
global viewport matrix sizes.
6. Human verification proves an owner can identify the exact missing gate,
review its evidence, request changes with a reason, and understand that a
Roadmap approval does not automatically start implementation.

### P-03.10 Standards References

- [WAI-ARIA Modal Dialog Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/): modal focus containment and return.
- [WAI-ARIA Alert Dialog Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/alertdialog/): high-consequence confirmation semantics.
- [W3C Complex Images](https://www.w3.org/WAI/tutorials/images/complex/):
  equivalent structured dependency information.
- [WAI-ARIA Grid Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/grid/):
  why the read-only Feature Unit list remains a native table.

---

## P-04: Feature Unit

### P-04.1 Identity And Purpose

| Property | Contract |
|---|---|
| Route | `/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}` |
| Read authority | matching nested `GET /v1/.../feature-units/{featureUnitKey}` endpoint |
| Primary question | Does this functional goal have approved scope, satisfied dependencies, complete Component Work evidence, and a clear next gate? |
| State-changing action | record a permitted human planning approval or change request |
| Non-goals | editing scope in place, starting implementation manually, direct Component Work commands, mutable acceptance criteria, PR merge, or treating checkboxes as state transitions |

A Feature Unit is the functional and human-verification unit. It is not a
Project phase, a branch, or a single Component Work. One Unit can require
server, web, mobile, design, and contract work. It reaches `ready_for_pr` only
when all required Component Works meet their independent gates.

### P-04.2 Read Model And API Shape

```text
GET /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}
```

The endpoint returns one bounded `FeatureUnitDetailProjection` with
`Cache-Control: private, no-store`. The API, not the browser, joins Feature
Unit, dependencies, Component Work, evidence, review, PR, and decision state.

```ts
interface FeatureUnitDetailResponse {
  featureUnit: {
    featureUnitKey: string
    sequenceNumber: number
    title: string
    intent: string
    state: string
    riskLevel: 'low' | 'normal' | 'high' | 'security_sensitive' | 'production_data_related'
    constraintProfile: { version: number; contentSha256: string; href: string }
    spec: {
      artifactKey: string
      revision: number
      contentSha256: string
      status: 'draft' | 'valid' | 'superseded' | 'rejected'
      href: string | null
    }
    resourceVersion: string
  }
  planningGate: {
    state: 'not_ready' | 'human_decision_required' | 'approved' | 'changes_requested' | 'blocked'
    explanationCode: string
    allowedActions: Array<'record_human_decision'>
    expectedResourceVersion: string
  }
  activationGate: {
    state: 'not_approved' | 'waiting_for_dependencies' | 'eligible' | 'active' | 'blocked'
    unmetDependencyCount: number
    waivedDependencyCount: number
    pauseOrIncidentBlocking: boolean
    explanationCode: string
  }
  acceptanceCriteria: {
    items: Array<{
      criterionKey: string
      description: string
      verificationMode: 'command' | 'artifact_review' | 'human_check' | 'mixed'
      required: boolean
      status: string
      evidenceHref: string | null
    }>
    totalCount: number
    requiredCount: number
  }
  dependencies: {
    prerequisites: Array<{
      featureUnitKey: string
      title: string
      relation: 'depends_on' | 'blocks'
      satisfied: boolean
      waivedByDecisionHref: string | null
      href: string
    }>
    dependents: Array<{
      featureUnitKey: string
      title: string
      relation: 'depends_on' | 'blocks'
      href: string
    }>
  }
  componentWork: {
    items: Array<{
      componentWorkKey: string
      title: string
      primaryComponent: { key: string; displayName: string }
      executionScope: 'single' | 'coordinated'
      scopeComponents: Array<{
        key: string
        displayName: string
        role: 'primary' | 'contributing' | 'shared_contract'
      }>
      required: boolean
      state: string
      verification: 'not_started' | 'running' | 'passed' | 'failed' | 'not_applicable'
      review: 'not_started' | 'running' | 'passed' | 'changes_requested' | 'human_required'
      pullRequest: { pullRequestId: string; status: string; href: string } | null
      attentionSeverity: AttentionSeverity
      href: string
    }>
    requiredCount: number
    optionalCount: number
    omittedCount: number
  }
  componentContracts: {
    items: Array<{
      contractKey: string
      title: string
      type: string
      status: string
      required: boolean
      producerComponent: string
      consumerComponent: string
      href: string | null
    }>
    omittedCount: number
  }
  humanVerification: {
    state: 'not_available' | 'pending' | 'in_progress' | 'passed' | 'failed'
    requiredItemCount: number
    passedRequiredItemCount: number
    failedRequiredItemCount: number
    href: string | null
  }
  timeline: {
    items: Array<{
      eventKey: string
      occurredAt: string
      actorLabel: string
      kind: 'state_transition' | 'verification' | 'review' | 'decision' | 'pull_request' | 'incident'
      fromState: string | null
      toState: string | null
      summary: string
      evidenceHref: string | null
    }>
    omittedCount: number
  }
  snapshot: { observedAt: string; requestId: string; resourceVersion: string }
}
```

The endpoint returns at most 30 criteria, 20 Component Works, 12 Component
Contracts, and 20 timeline events. It reports omission counts and supplies a
dedicated detail route rather than silently discarding records. It never
returns raw ContextPackets, prompts, provider output, unredacted logs, or a
direct database state field.

### P-04.3 Human Planning Decision

Only `ready_for_human_review` can expose the decision action. The API reports
eligibility through `planningGate.allowedActions` and accepts:

```text
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/commands/record-human-decision
```

```ts
interface RecordFeatureUnitPlanningDecisionRequest {
  decision: 'approved' | 'changes_requested'
  expectedResourceVersion: string
  reason?: string
}
```

Approval creates `HumanDecision feature_unit_approved`; later activation is
automatic only after all dependency and safety gates pass. A change request
requires a trimmed 10 to 2,000 character reason, creates
`feature_unit_changes_requested`, returns the Unit to `draft`, and requires a
new FeatureUnitSpec revision and review packet. It never overwrites the old
Spec or reuses the old review packet as approval evidence.

The action lives in a review panel containing the current Spec revision/hash,
criteria, required Component mapping, dependencies, risk reason, and planning
questions. The dialog sends an `Idempotency-Key`; a `409` preserves the typed
reason but blocks resubmission until a fresh detail snapshot is loaded. The
browser cannot activate a Feature Unit or directly change its state.

### P-04.4 Layout And Section Contracts

At `1440px` and above:

```text
breadcrumb
identity: sequence, title, state, risk, snapshot
planning/activation gate bands when unresolved
scope and acceptance criteria (7 columns) | dependency condition (5 columns)
Component Work matrix (full width)
Component Contracts (7 columns) | human verification summary (5 columns)
append-only timeline (full width)
```

The header includes Roadmap context and immutable constraint-profile version.
It does not imply that a title is approved scope. Gate bands distinguish:

```text
Planning gate: may this Unit be approved as intended scope?
Activation gate: may approved work start without violating dependency or safety policy?
```

An approved Unit waiting on a server dependency is therefore neither failed nor
ready to implement; it is explicitly `waiting_for_dependencies`.

Acceptance criteria use a native table at `1024px` and above, then semantic
cards. Columns are `Requirement`, `Required`, `Verification mode`, `Evidence
state`, and `Evidence`. The page does not show a percentage complete: command,
artifact, and human checks have non-equivalent gates. It shows factual counts
and each criterion's state instead.

Dependencies are two native lists, `선행 조건` and `이 작업을 기다리는 단위`.
Each prerequisite says satisfied, waived with decision link, or unmet. A
supplementary diagram may follow P-02's SVG-plus-text contract, but the lists
are primary and the only representation below `768px`.

The Component Work table uses `Component / Work`, `Scope`, `Required`,
`State`, `Verification`, `Review`, `PR`, and `Open` columns. Scope describes
coordinated participating Components in text. Rows never contain pause, retry,
or cancel: those commands require Component Work detail context. Component
Contracts render separately; an unaccepted required contract blocks the
relevant Work from `ready_for_pr` and must be displayed with producer,
consumer, requirement, state, and proof link.

### P-04.5 Lifecycle, Events, And State Matrix

The timeline is a native ordered list of immutable state/evidence events. It
shows `from -> to` only for state transitions; other entries state which type
of evidence changed. It is not an editable workflow diagram.

After initial REST success, the page subscribes to the Project SSE. New events
increment one pending-update counter. They do not mutate the Work table,
timeline, criteria, or gate currently under keyboard focus. `업데이트 적용`
fetches and atomically replaces the complete Feature Unit projection. New
incidents and required human decisions also render a persistent, non-sensitive
gate notice.

| Condition | Required presentation |
|---|---|
| `draft` | current Spec revision and remaining review-readiness evidence; no decision control |
| `ready_for_human_review` | review panel with both permitted human decisions |
| `approved` with unmet dependency | approved scope plus prerequisite links and activation wait condition |
| `active` | current Component Work states and downstream links |
| `implementation_done` | implementation evidence; verification/review is next |
| `verification_running` / `review_running` | factual running state and Run/Review links; no invented percent progress |
| `needs_revision` | failed verification or accepted finding and impacted Work links |
| `ready_for_pr` / `pr_created` | PR/evidence summary, never a merge button |
| `human_verification_pending` | checklist summary and verification route link |
| `human_verified` / `closed` | immutable evidence summary and read-only navigation |
| `blocked` / `incident_hold` / `cancelled` | proving event and recovery/incident context; no bypass control |

### P-04.6 Responsive And Accessibility Contract

| Range | Required adaptation |
|---|---|
| 1280px+ | full criteria and Component Work tables; two-column scope/dependency and contract/verification rows |
| 1024-1279px | compact tables; stack panels below their `420px` container threshold |
| 768-1023px | criteria and Work cards; contracts/verification single column; dependency lists only |
| 320-767px | single-column sections; header metadata in disclosure rows; every action stays a labelled text button |

The route has one `main` landmark and a Feature Unit `h1`. Gate bands are
labelled sections before affected content. Tables retain native captions,
headers, and links; no ARIA grid is used. Status color is always paired with
text/icon. SSE never moves focus. Long keys and paths use the global
copy-and-tooltip contract.

### P-04.7 Planned Frontend Boundaries

```text
apps/control/app/(control)/projects/[projectKey]/roadmaps/[roadmapKey]/feature-units/[featureUnitKey]/page.tsx
apps/control/features/feature-units/feature-unit-detail-route.tsx
apps/control/features/feature-units/feature-unit-gate-band.tsx
apps/control/features/feature-units/feature-unit-planning-decision-dialog.tsx
apps/control/features/feature-units/acceptance-criteria-table.tsx
apps/control/features/feature-units/acceptance-criteria-cards.tsx
apps/control/features/feature-units/feature-dependency-lists.tsx
apps/control/features/feature-units/component-work-matrix.tsx
apps/control/features/feature-units/component-contract-list.tsx
apps/control/features/feature-units/human-verification-summary.tsx
apps/control/features/feature-units/feature-unit-timeline.tsx
packages/contracts/src/feature-units/feature-unit-detail.contract.ts
packages/contracts/src/feature-units/feature-unit-commands.contract.ts
```

The generated client resolves the nested route and typed errors. The route
feature owns one snapshot reducer and Project SSE freshness subscription. Each
section is a typed renderer; it cannot calculate a transition, derive Work
readiness, or write a HumanDecision.

### P-04.8 Verification Contract

1. State-machine/application tests prove `ready_for_human_review -> draft`
requires a reason and new Spec revision, and old review evidence cannot be
reused.
2. API tests prove nested addressing rejects mismatched Project, Roadmap,
Feature Unit, and Component Work combinations without disclosure.
3. Projection tests prove gate values, dependency waiver display,
required-versus-optional Work handling, contract blockers, and no synthetic
criterion percentage.
4. UI tests prove decision-dialog conflict recovery, semantic table/card
switching, stable SSE refresh, and every lifecycle condition above.
5. Accessibility and visual checks cover long Korean copy, 30 criteria, 20
Works, coordinated scope labels, unmet dependency, blocked state, and each
global viewport.
6. Human verification proves that an owner can determine why a Unit is not
active or not PR-ready without a raw log or color-only inference.

### P-04.9 Standards References

- [WAI-ARIA Modal Dialog Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/): planning decision dialog behavior.
- [WAI-ARIA Grid Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/grid/): native table is preferred for non-editable data.
- [W3C Complex Images](https://www.w3.org/WAI/tutorials/images/complex/): dependency diagrams need equivalent structured relationships.

---

## P-05: Component Work

### P-05.1 Identity And Safety Boundary

| Property | Contract |
|---|---|
| Route | `/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}` |
| Primary question | What exact implementation/PR unit is this, what evidence has it produced, and which server-approved intervention is currently safe? |
| Read authority | matching nested `GET /v1/.../component-works/{componentWorkKey}` endpoint |
| Commands | pause, resume, retry, cancel, request PR; only when returned in `allowedActions` |
| Non-goals | terminal access, arbitrary command execution, direct branch checkout, worktree-path disclosure, direct status writes, PR merge, or retrying a failed attempt without recovery evidence |

Component Work is the one branch/one PR execution unit. A `single` Work has one
primary Component root. A `coordinated` Work has one primary root plus declared
contributing/shared-contract roots, but still one branch and one PR. The page
must make that atomic boundary visible before it exposes any command.

### P-05.2 Detail Projection

```text
GET /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}
```

The response is a no-store bounded projection. Absolute worktree paths,
environment variables, provider credentials, raw command argv, raw logs, and
unredacted artifact content never appear.

```ts
interface ComponentWorkDetailResponse {
  work: {
    componentWorkKey: string
    title: string
    intent: string
    state: string
    riskLevel: string
    executionScope: 'single' | 'coordinated'
    required: boolean
    primaryComponent: { key: string; displayName: string }
    scopes: Array<{
      componentKey: string
      displayName: string
      relativeRoot: string
      role: 'primary' | 'contributing' | 'shared_contract'
      required: boolean
    }>
    repository: {
      remoteUrlRedacted: string
      integrationBranch: string
      baseBranch: string
      headBranch: string | null
    }
    worktree: { worktreeKey: string; status: string; baseCommit: string | null; headCommit: string | null } | null
    allowedPaths: { version: string; writeRuleCount: number; href: string | null }
    resourceVersion: string
  }
  commandGate: {
    paused: boolean
    pauseReason: string | null
    allowedActions: Array<'pause' | 'resume' | 'retry' | 'cancel' | 'request_pr'>
    actionRequirements: Record<string, {
      expectedResourceVersion: string
      reasonRequired: boolean
      confirmation: 'none' | 'dialog' | 'destructive_dialog'
    }>
  }
  activeExecution: {
    jobAttemptId: string | null
    jobKey: string | null
    state: 'none' | 'queued' | 'leased' | 'running' | 'timed_out' | 'failed' | 'human_required'
    runnerLabel: string | null
    startedAt: string | null
    timeoutAt: string | null
    lastHeartbeatAt: string | null
    href: string | null
  }
  verification: {
    state: 'not_started' | 'running' | 'passed' | 'failed' | 'blocked'
    latestVerificationRunId: string | null
    requiredCommandCount: number
    passedCommandCount: number
    failedCommandCount: number
    href: string | null
  }
  review: {
    state: 'not_started' | 'local_running' | 'local_completed' | 'arbiter_running' | 'passed' | 'changes_requested' | 'human_required'
    reviewGroupId: string | null
    acceptedP0P1FindingCount: number
    unresolvedFindingCount: number
    href: string | null
  }
  pullRequest: {
    pullRequestId: string
    url: string
    baseBranch: string
    headBranch: string
    status: string
    createdAt: string
  } | null
  attempts: {
    items: Array<{
      jobAttemptId: string
      attemptNumber: number
      state: string
      workerLabel: string | null
      startedAt: string | null
      finishedAt: string | null
      timeoutAt: string | null
      failureCode: string | null
      redactedSummary: string | null
      href: string
    }>
    omittedCount: number
  }
  timeline: { items: Array<{ eventKey: string; occurredAt: string; summary: string; evidenceHref: string | null }>; omittedCount: number }
  snapshot: { observedAt: string; requestId: string; resourceVersion: string }
}
```

`relativeRoot` is repository-relative, normalized, and allowed only because it
is part of the approved ComponentRepository scope. Local absolute paths remain
Worker-only. `baseBranch` must be `integrate`; any projection that reports
another base is an error/incident condition, not a selectable value.

### P-05.3 Command Contract

Every command uses the nested route, `Idempotency-Key`, expected resource
version, PolicyDecision, EvidenceGate, StateMachine, AuditEvent, and durable
response reference. The browser submits no shell, path, branch, model, or
command-line input.

```text
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}/commands/pause
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}/commands/resume
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}/commands/retry
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}/commands/cancel
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/component-works/{componentWorkKey}/commands/request-pr
```

All command bodies have `expectedResourceVersion` and `reason`. `reason` is
required and 10 to 2,000 characters for pause, resume, retry, and cancel;
request PR accepts an optional 2,000-character note. `cancel` is destructive:
its alert dialog names the Work key, current attempt, irreversible effects, and
the required reason. Initial focus is `취소`, not the destructive action.

`pause` creates a PauseRecord and does not counterfeit a Component Work state.
`resume` records the required HumanDecision and only schedules new work after
policy allows it. `retry` can create a new JobAttempt only after retry policy
and side-effect recovery evidence pass; it never rewrites a terminal attempt.
`request_pr` can enqueue a PR job only when server evidence proves
`ready_for_pr`; it creates a PR but never merges it.

`409` preserves the form and requires a fresh snapshot. `422` displays the
stable policy/evidence reason and link when authorized. A `202` result is
rendered as `요청됨`, never as a completed pause/retry/PR.

### P-05.4 Layout, Evidence, And Live State

At `1440px` and above:

```text
breadcrumb
identity: Work key, state, scope, required flag, snapshot
command strip: pause flag and only server-authorized controls
scope/repository/branch panel (7 columns) | active execution panel (5 columns)
verification panel (6 columns) | review panel (6 columns)
attempt history table (full width)
PR panel (when present) and append-only timeline
```

The scope panel lists every participating Component and role. Repository data
shows redacted remote URL, integration branch, base branch, head branch, commit
identifiers, worktree key/status, and allowed-path rule version/count. It never
shows a local path as a clickable filesystem affordance.

The active-execution panel distinguishes `queued`, `leased`, `running`,
`timed_out`, `failed`, and `human_required`. Elapsed time is not a progress
bar. Near-timeout begins below 20 percent remaining and shows the exact deadline.
Verification displays command counts and latest run link; Review displays local
council/arbiter state plus accepted P0/P1 and unresolved counts. Neither panel
claims success from an Agent's self-report.

Attempts use a native table with attempt number, state, Worker, start/end,
deadline, redacted failure code/summary, and Run Detail link. It keeps at most
20 records with an explicit history route. The timeline is a native ordered
list; it does not stream raw output.

Project SSE marks the page stale and increments pending updates. It never
replaces command eligibility or attempt rows during a focused confirmation
dialog. After a command response or reconnect, the page re-fetches one complete
snapshot before enabling new commands.

### P-05.5 Responsive, Accessibility, And Verification

At `1280px+`, attempts are a full table. At `1024-1279px`, columns collapse to
attempt/state/time/summary/open. Below `1024px`, attempts are cards and scope
roles are a labelled list. Below `768px`, command controls become a vertical
group above all evidence; no destructive action enters a fixed bottom bar.

The command strip is a labelled toolbar only when two or more controls are
available; otherwise it is ordinary document-order buttons. Each control has
an explicit accessible label, disabled reason, and result announcement. Dialogs
follow P-03 focus rules. Keyboard focus never enters an SVG, log, or hidden
attempt row. All status color has text/icon redundancy.

Acceptance requires controller/OpenAPI validation of every body and nested key;
integration proof that pause is a PauseRecord, retry creates a new attempt,
PR target is `integrate`, and no command bypasses policy/evidence; UI tests for
all `200/202/409/422` command results and focus restoration; visual checks for
single/coordinated scope, active timeout, failed verification, review changes,
paused, incident hold, PR created, and long branch/path values; and a human
check that no control can be mistaken for a merge, terminal, or direct state
edit.

### P-05.6 Planned Frontend Boundaries

```text
apps/control/app/(control)/projects/[projectKey]/roadmaps/[roadmapKey]/feature-units/[featureUnitKey]/component-works/[componentWorkKey]/page.tsx
apps/control/features/component-work/component-work-detail-route.tsx
apps/control/features/component-work/component-work-command-toolbar.tsx
apps/control/features/component-work/component-work-command-dialog.tsx
apps/control/features/component-work/component-work-scope-panel.tsx
apps/control/features/component-work/active-execution-panel.tsx
apps/control/features/component-work/verification-summary-panel.tsx
apps/control/features/component-work/review-summary-panel.tsx
apps/control/features/component-work/job-attempt-history.tsx
apps/control/features/component-work/component-work-timeline.tsx
packages/contracts/src/component-work/component-work-detail.contract.ts
packages/contracts/src/component-work/component-work-commands.contract.ts
```

### P-05.7 Standards References

- [WAI-ARIA Alert Dialog Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/alertdialog/): destructive cancel confirmation.
- [WAI-ARIA Modal Dialog Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/): focus containment and return.
- [WAI-ARIA Grid Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/grid/): attempts remain native data tables, not composite grids.

---

## P-06: Run Detail And Logs

### P-06.1 Identity And Read Boundary

| Property | Contract |
|---|---|
| Route | `/runs/{jobAttemptId}` where `jobAttemptId` is a global UUID |
| Primary question | What immutable attempt ran, under which authorized context, what happened, and what evidence proves its terminal or current state? |
| Read authority | `GET /v1/job-attempts/{jobAttemptId}` plus cursor-paginated `GET /v1/logs?attemptId={jobAttemptId}` |
| Commands | none; retry/pause/cancel live only on Component Work detail |
| Non-goals | terminal emulation, command input, provider prompt editing, log export by default, raw payload rendering, automatic tail-follow, or reinterpretation of an exit code as Component Work success |

The JobAttempt is immutable execution history. A retry creates another attempt;
it never mutates this page's attempt. The page can link to the logical Job and
Component Work, but it does not infer that a succeeded AgentRun means the
Feature Unit or Component Work is complete.

### P-06.2 DTO And Pagination Contract

```ts
interface JobAttemptDetailResponse {
  attempt: {
    jobAttemptId: string
    attemptNumber: number
    state: 'leased' | 'running' | 'succeeded' | 'failed' | 'timed_out' | 'cancelled' | 'policy_denied' | 'blocked' | 'human_required'
    job: { jobKey: string; type: string; targetHref: string }
    worker: { workerKey: string; version: string; href: string | null } | null
    lease: { leasedAt: string | null; expiresAt: string | null; lastHeartbeatAt: string | null }
    timing: { startedAt: string | null; finishedAt: string | null; timeoutAt: string | null; durationMs: number | null }
    terminal: { exitCode: number | null; signal: string | null; failureCode: string | null; redactedSummary: string | null }
    resultArtifactHref: string | null
    replacementAttemptHref: string | null
    resourceVersion: string
  }
  agentRuns: Array<{ agentRunId: string; role: string; provider: string; modelIdentifier: string; state: string; startedAt: string; finishedAt: string | null; outputArtifactHref: string | null; href: string }>
  commandRuns: Array<{ commandRunId: string; commandKey: string; state: string; startedAt: string; finishedAt: string | null; exitCode: number | null; timedOut: boolean; stdoutAvailable: boolean; stderrAvailable: boolean; href: string }>
  artifacts: Array<{ artifactKey: string; type: string; status: string; redactionStatus: string; byteSize: number; href: string | null }>
  snapshot: { observedAt: string; requestId: string; resourceVersion: string }
}

interface AttemptLogPageResponse {
  attemptId: string
  stream: 'stdout' | 'stderr' | 'system'
  entries: Array<{ sequence: number; occurredAt: string; level: 'info' | 'warn' | 'error'; text: string }>
  nextCursor: string | null
  newestSequence: number
  redaction: { applied: boolean; omittedEntryCount: number; reasonCode: string | null }
}
```

Log entries are sanitized server-side and returned as plain text. The UI inserts
them with text nodes, never `dangerouslySetInnerHTML` or rendered ANSI/HTML.
Each request limits entries to 500 and total returned text to 1 MiB; an opaque
cursor is the only pagination mechanism. `stdout`, `stderr`, and `system` are
separate server streams so one cannot be mistaken for another.

### P-06.3 Layout And Log Interaction

At wide desktop the route has: identity/terminal summary; timing and worker
facts; AgentRun and CommandRun tables; redacted Artifact list; then a log
reader. The terminal summary uses explicit text such as `timed_out`, exit code,
signal, failure code, and result Artifact link. It never labels a nonzero exit
as a vague red success/failure pill.

The log reader uses URL state `?stream=stdout|stderr|system&cursor=...` and
shows one stream at a time. It is a manual-activation tab interface only after
all three first pages are preloaded; otherwise use ordinary links to preserve
predictable latency. The raw output region is a labelled `<pre>` inside a
bounded scroll container, preserves whitespace, wraps only when the owner
explicitly enables `줄 바꿈`, and offers copy-visible-text only. It is not an
ARIA live log region because continuous token output would overwhelm assistive
technology.

```text
SSE log.available
-> record highest newestSequence only
-> show "새 출력 있음" through role=status
-> do not append or scroll
-> owner selects "새 출력 가져오기"
-> request after current newest cursor; preserve scroll position
```

When the user is at the visual bottom and explicitly selects new output, the
reader may scroll to the first new entry, never to the absolute bottom. When
not at bottom, it retains viewport and displays an anchored new-output marker.
Long unbroken values use local horizontal scroll; the page does not horizontal
scroll. Redacted/omitted entries show count and reason, not placeholder text
pretending the original output exists.

### P-06.4 States, Focus, And Verification

| Condition | Required presentation |
|---|---|
| leased/running | current lease, heartbeat freshness, timeout deadline, and no completion claim |
| succeeded | attempt result only; link to verification/review evidence before any readiness conclusion |
| failed/timed out | failure code, redacted summary, replacement attempt if any, Component Work link for permitted recovery |
| policy denied/blocked/human required | policy reason/evidence link when authorized; no retry control here |
| no AgentRun | explicit `AgentRun 없음`; never fabricate model/provider data |
| no log page | neutral empty output state by stream |
| redacted output | redaction fact/count/reason; no client-side reveal affordance |
| stale/disconnected | retained snapshot and non-live marker; reconnect requires REST refetch |
| attempt unavailable | clear cached data and show non-discoverable result |

The `h1` names the JobAttempt and state. Tables are native tables. Stream
links/buttons retain normal Tab order. New-output status uses `role=status`
and does not steal focus. Artifact links describe type and redaction status.
Verification requires controller tests for cursor bounds/redaction, integration
tests proving no raw content reaches SSE, UI tests for no auto-scroll and
scroll preservation, security tests for text-only insertion, and visual checks
for running, timeout, redacted stderr, empty stream, and long-line overflow.

### P-06.5 Planned Frontend Boundaries

```text
apps/control/app/(control)/runs/[jobAttemptId]/page.tsx
apps/control/features/runs/job-attempt-detail-route.tsx
apps/control/features/runs/attempt-summary.tsx
apps/control/features/runs/agent-run-table.tsx
apps/control/features/runs/command-run-table.tsx
apps/control/features/runs/redacted-artifact-list.tsx
apps/control/features/runs/attempt-log-reader.tsx
apps/control/features/runs/attempt-log-state.ts
packages/contracts/src/runs/job-attempt-detail.contract.ts
packages/contracts/src/runs/attempt-log.contract.ts
```

### P-06.6 Standards References

- [WAI-ARIA Tabs Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/tabs/): manual activation when panel load latency is material.
- [WAI-ARIA Table Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/table/): native tables for static run facts.
- [W3C ARIA status technique](https://www.w3.org/WAI/WCAG20/Techniques/aria/ARIA22): noninterrupting new-output notices.

---

## P-07: Verification And Review Evidence

### P-07.1 Routes And Questions

| Route | Primary question | Authority |
|---|---|---|
| `/verification-runs/{verificationRunId}` | Did required deterministic verification run against the exact Work and Spec revision? | `GET /v1/verification-runs/{verificationRunId}` |
| `/reviews/{reviewGroupId}` | What did each reviewer find, and what did the Arbiter decide? | `GET /v1/review-groups/{reviewGroupId}` |

Both UUID routes are read-only evidence views. They cannot mark a test passed,
silence a finding, edit a reviewer result, or create a PR. That preserves the
separation between implementation, verification, independent review, and
arbitration.

### P-07.2 Verification Run Contract

The response binds Component Work route, git snapshot commit/tree/base IDs,
VerificationProfile key/version, effective Spec Library revision/manifest hash,
start/end/failure details, `requiredPassed`, EvidenceGate result, and commands.
Every command has its `commandRunId`, key, required flag, state, expected exit
codes, actual exit/signal/timeout, duration, redacted output links, and
authorized evidence links. If profile, snapshot, or Spec revision does not
match evaluated Work, the page renders `evidence_invalid`, never a pass.

Commands use a native table at desktop and cards below `1024px`. `not_started`,
`running`, `passed`, `failed`, `timed_out`, `blocked`, and `evidence_invalid`
are distinct. A green command never becomes an overall claim: the stored
`requiredPassed` and its EvidenceGate reference are the only page conclusion.

### P-07.3 Review Group Contract

The response includes review packet/hash, risk level, required reviewer count,
group state, reviewer results, findings, server deduplication links,
ArbiterDecision, and RevisionTasks. A reviewer result identifies reviewer key,
provider/model, runner-local status, completed time, and result Artifact link.
It never exposes private reasoning or unredacted raw output.

Findings use a native table: severity, category, title, authorized file/line,
source reviewer count, resolution, evidence, and RevisionTask. The API groups
findings by `deduplication_hash`; the browser never decides duplicates. Default
sort is accepted P0/P1, accepted P2, human-required, then informational.

The Arbiter panel renders one active decision:

```text
ready_for_pr | needs_revision | human_required | blocked
```

It includes decision Artifact, timestamp, summary, unresolved accepted finding
count, and whether PR creation is permitted. `ready_for_pr` permits the next
evidence gate only; it is not a GitHub merge approval.

### P-07.4 UI And Evidence Rules

Verification layout is identity/evidence binding, required-result banner,
command results, artifacts. Review layout is group identity, reviewer coverage,
Arbiter panel, findings, RevisionTasks. Both use semantic headings/native
tables and one `role=status` freshness notice. Committed SSE changes do not
reorder a focused table; `업데이트 적용` refetches the full projection.

Failed, timeout, partial review, missing reviewer, mismatched evidence,
redacted artifact, stale snapshot, and denied states show a visible reason and
authorized evidence link. There is no `ignore finding` action. Human-required
routes to Decision Inbox; revision-required routes to RevisionTask/Work.

### P-07.5 Verification And Files

Tests prove UUID addressing, redaction, profile/snapshot/spec mismatch
rejection, risk-based reviewer-count enforcement, P0/P1 PR blocking,
server-owned deduplication, no automatic reorder, and all terminal states.
Human verification proves reviewer disagreement is never mistaken for an
Arbiter decision.

```text
apps/control/app/(control)/verification-runs/[verificationRunId]/page.tsx
apps/control/app/(control)/reviews/[reviewGroupId]/page.tsx
apps/control/features/verification/verification-run-detail.tsx
apps/control/features/verification/verification-command-results.tsx
apps/control/features/reviews/review-group-detail.tsx
apps/control/features/reviews/reviewer-coverage.tsx
apps/control/features/reviews/arbiter-decision-panel.tsx
apps/control/features/reviews/review-findings-table.tsx
apps/control/features/reviews/revision-task-list.tsx
packages/contracts/src/verification/verification-run.contract.ts
packages/contracts/src/reviews/review-group.contract.ts
```

### P-07.6 Standards References

- [WAI-ARIA Table Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/table/): static evidence tables retain native semantics.
- [W3C ARIA status technique](https://www.w3.org/WAI/WCAG20/Techniques/aria/ARIA22): noninterrupting committed-update notices.

---

## P-08: Pull Request And Human Verification

### P-08.1 Boundaries

| Surface | Route | Primary question |
|---|---|---|
| Pull Request detail | `/pull-requests/{pullRequestId}` | Was this PR created from the correct Work, branch, and evidence, and is it ready for a human to inspect? |
| Human verification | `/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/human-verification` | Has the human performed the required Feature Unit checks and recorded durable evidence before closure? |

ADO creates PRs with base branch `integrate`; it never renders a merge command,
merge-ready status, GitHub review approval, or deployment as an ADO action.
Human verification is Feature Unit scoped because cross-component functionality
cannot be proven by a single Component Work PR.

### P-08.2 PR Detail Contract

`GET /v1/pull-requests/{pullRequestId}` returns the UUID-addressed PR,
Component Work route, provider/external number/url, base/head branch, created
and last-synced time, PR status projection, PullRequestPacket Artifact,
effective Spec revision/hash, latest GitSnapshot, verification/review gate
summaries, and redacted changed-path summary. It verifies base branch is
`integrate`; any other base produces a policy/error panel rather than a normal
PR view.

The screen order is identity and external GitHub link; immutable branch/snapshot
binding; readiness evidence; changed-path summary; linked verification/review;
and sync timeline. Changed paths are summaries/authorized links, not an inline
unbounded diff viewer. The external URL opens GitHub in a new tab with a clear
label. Browser rendering never attempts to reproduce GitHub merge controls.

### P-08.3 Human Verification Read And Command Contract

```text
GET /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/human-verification
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/human-verification-items/{itemKey}/commands/record-result
POST /v1/projects/{projectKey}/roadmaps/{roadmapKey}/feature-units/{featureUnitKey}/commands/record-human-verification-decision
```

The read projection contains Feature Unit state/gate, all required PR links,
checklist item key/title/instructions/required flag/current latest append-only
result/evidence link, and final decision eligibility. It does not return test
credentials, private data, or an editable checklist template.

```ts
interface RecordHumanVerificationItemRequest {
  result: 'passed' | 'failed' | 'skipped'
  expectedFeatureUnitVersion: string
  reason?: string
  evidenceArtifactKey?: string
}

interface RecordHumanVerificationDecisionRequest {
  decision: 'human_verified' | 'changes_requested'
  expectedFeatureUnitVersion: string
  reason?: string
}
```

`failed`, `skipped`, and final `changes_requested` require a 10 to 2,000
character reason. `skipped` additionally requires a policy-permitted explicit
HumanDecision. `human_verified` is enabled only when the API reports all
required items passed or validly skipped, all required PR evidence is visible,
and no current incident/pause blocks closure. A result appends evidence; it
never edits a prior result in place. The final decision is not enabled based on
client checkbox counts.

### P-08.4 Layout And Interaction

PR detail uses fact panels and native evidence tables. Human verification uses:

```text
Feature Unit and PR evidence header
gate explanation / final-decision eligibility
required checklist table
optional checklist disclosure
final human decision panel
append-only verification-result timeline
```

Each required checklist row shows instruction, current result, evidence, last
recorded time, and `결과 기록` button only when API authorizes it. Recording
opens a labelled dialog with result radios, reason field that appears/validates
when required, optional evidence Artifact selector limited to authorized same
Project artifacts, and clear statement that the result is append-only. Final
verification uses an alertdialog, initially focused on `취소`, with the exact
required-item counts and PR links visible before confirmation.

At desktop checklist is a native table; below `1024px` it becomes labelled
cards; below `768px` each record action is full-width but the final destructive
or consequential decision remains separate below all evidence. SSE creates a
pending update marker only. It never changes a selected radio, typed reason,
or checklist row during a dialog.

### P-08.5 States And Verification

| Condition | Required rendering |
|---|---|
| PR absent | clear evidence that Work is not yet PR-created; no fake external link |
| PR created, human verification unavailable | PR facts plus Feature Unit state explaining why checklist is not open |
| pending checklist | required/optional separation, current item evidence, final action disabled with reason |
| failed item | failure reason/evidence, Component Work and revision context link |
| skipped item | explicit skip reason and HumanDecision evidence |
| all items eligible | final human decision panel enabled by server projection only |
| human verified | immutable result summary; merge remains external/human |
| changes requested | reason and routed revision context; no silent reopen |
| stale/conflict | preserve entered form, require REST refresh for versioned resubmit |

Tests cover nested addressing, append-only result history, reason/skip policy,
final-gate enforcement, no merge endpoint/UI, dialog focus, mobile cards,
SSE form stability, and denial/redaction. Human acceptance proves the owner can
open every required PR, execute each checklist instruction, attach permitted
evidence, and record a final result without mistaking it for a Git merge.

### P-08.6 Planned Frontend Boundaries

```text
apps/control/app/(control)/pull-requests/[pullRequestId]/page.tsx
apps/control/app/(control)/projects/[projectKey]/roadmaps/[roadmapKey]/feature-units/[featureUnitKey]/human-verification/page.tsx
apps/control/features/pull-requests/pull-request-detail.tsx
apps/control/features/human-verification/human-verification-route.tsx
apps/control/features/human-verification/checklist-table.tsx
apps/control/features/human-verification/checklist-cards.tsx
apps/control/features/human-verification/record-item-result-dialog.tsx
apps/control/features/human-verification/final-verification-dialog.tsx
packages/contracts/src/pull-requests/pull-request.contract.ts
packages/contracts/src/human-verification/human-verification.contract.ts
```

### P-08.7 Standards References

- [WAI-ARIA Alert Dialog Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/alertdialog/): final consequential human decision.
- [WAI-ARIA Modal Dialog Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/): result-recording dialog focus behavior.
- [WAI-ARIA Table Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/table/): checklist evidence table semantics.

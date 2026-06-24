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

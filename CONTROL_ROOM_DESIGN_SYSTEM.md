# ADO Control Room Design System

This is the visual and interaction contract for ADO's own Control Room in
`apps/control`. It does not apply to managed Project products. Project design
governance is defined in `PROJECT_DESIGN_GOVERNANCE.md`.

## 1. Product Character

The Control Room is a Korean-first, desktop-first operating console for a
human who supervises automated development. It must be calm, dense, explicit,
and trustworthy. It is not a marketing site, generic admin template, terminal
emulator, or database browser.

The UI reports committed system state. It never implies a successful action
before the API has accepted and the underlying state is committed.

## 2. Layout Contract

| Viewport range | Grid | Page inset | Navigation |
|---|---:|---:|---|
| 320-479 px | 4 columns | 16 px | modal Drawer |
| 480-767 px | 4 columns | 16 px | modal Drawer |
| 768-1023 px | 6 columns | 20 px | 64 px icon rail |
| 1024-1439 px | 8 columns | 24 px | 256 px sidebar |
| 1440 px and above | 12 columns | 32 px | 256 px sidebar |

- Top bar height is `56px`.
- Desktop sidebar width is `256px`; compact rail width is `64px`.
- Content maximum width is `1600px`.
- The only spacing values are `0, 4, 8, 12, 16, 20, 24, 32, 40, 48, 64px`.
- Grid items use `min-width: 0`; no global page-level horizontal overflow.
- Logs, code, long identifiers, paths, and audit tables may scroll only inside
  their own bounded region.

At less than `1024px`, the sidebar becomes an icon rail. At less than `768px`,
the rail becomes a Drawer. Summary grids use four columns at wide desktop, two
at desktop/tablet where space permits, and one below `768px`.

## 3. Type And Content

Primary UI language is Korean. Machine values remain in their original English
form and use monospace.

```text
Sans: Pretendard Variable, Pretendard, Noto Sans KR, system-ui
Mono: JetBrains Mono, ui-monospace, SFMono-Regular, Menlo, monospace
```

| Use | Size / line height | Weight |
|---|---|---:|
| page title | 24 / 32 px | 650 |
| section title | 16 / 24 px | 650 |
| body | 14 / 20 px | 400 |
| dense body | 13 / 18 px | 400 |
| label and status | 12 / 16 px | 600 |
| code and logs | 12 / 18 px | 450 |

Korean prose uses `word-break: keep-all`. IDs, branches, paths, and SHA values
truncate to one line with copy action and tooltip. Log and JSON views preserve
lines and use local horizontal scrolling. Times show relative text by default
and an absolute timestamp with time zone on hover and focus.

## 4. Color And State Tokens

| Role | Token |
|---|---|
| canvas | `#F8FAFC` |
| surface | `#FFFFFF` |
| muted surface | `#F1F5F9` |
| strong text | `#0F172A` |
| secondary text | `#475569` |
| tertiary text | `#64748B` |
| primary action | `#1D4ED8` |
| running | `#0F766E` |
| success | `#047857` |
| warning / human action | `#92400E` |
| failure / destructive | `#B91C1C` |

Normal text must meet a 4.5:1 contrast ratio against its rendered background.
State is always icon plus Korean label plus, where useful, a reason summary;
color alone is never a state carrier.

Canonical states are `대기`, `실행 중`, `차단됨`, `검토 필요`, `실패`, `완료`,
`검증 완료`, and `취소됨`. Their machine values come only from API state
projections; the client does not invent local success states.

## 5. Components And Input

- Panels: `1px solid #E2E8F0`, `6px` radius, `16px` padding; dense detail
  panels may use `20px` padding.
- General cards use no shadow. Surface, border, and spacing provide hierarchy.
- Every interactive target has a minimum `44 x 44px` hit area. Its desktop
  visual control may be `36px` high within that target.
- Interactive rows have a minimum `44px` height. Passive data rows have a
  minimum `40px` height.
- Focus is a `2px #1D4ED8` outline with `2px` offset and is never removed.
- Icon-only controls require an accessible name and tooltip.
- Status badges are descriptive, not buttons.
- Pause, retry, cancel, reject, and destructive actions use text labels; an
  icon alone is insufficient.
- Disabled controls expose their reason in adjacent help text or a tooltip.

## 6. Live State, Commands, And Failure

SSE is an update hint, not the source of truth. A state-changing command shows
`requested`, then API-confirmed `accepted`, `rejected`, or `conflicted`.

- New events update the affected row without auto-reordering the user's list.
- Reordering changes display `새 업데이트 N건`; the user chooses when to apply.
- A changed row receives an `업데이트됨` marker for eight seconds.
- Long operations distinguish queued/requested from runner-owned/running.
- Retries show attempt number, latest failure reason, and next scheduled time.
- Blocked work shows the blocker, release condition, and responsible actor.
- Every list and detail view supplies loading, empty, error, permission-denied,
  stale, and disconnected states.

## 7. Motion And Accessibility

Allowed durations are `120ms`, `160ms`, and `200ms` using
`cubic-bezier(0.2, 0, 0, 1)`. Only opacity, color, and small outline changes
may transition. Layout-shifting animation, counter animation, infinite pulse,
and infinite shimmer are prohibited.

When `prefers-reduced-motion: reduce` is active, nonessential transitions and
shimmer are removed. Primary user paths must be keyboard-operable, retain
visible focus, and provide semantic labels and errors.

## 8. Visual QA Matrix

Every core screen is inspected at:

```text
320x568, 390x844, 768x1024, 1024x768, 1440x900, 1920x1080
```

Acceptance requires no clipping, overlap, inaccessible focus, unintended page
horizontal scroll, or state contradiction. Loading, empty, unauthorized,
network failure, SSE reconnect, and execution failure states are verified when
applicable. Visual QA is evidence for the ADO Control Room only; Project UIs
use their own approved responsive matrix and acceptance checklist.

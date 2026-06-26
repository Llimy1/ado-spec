# ADO Control Room Design Tokens

This document makes the ADO Control Room design system implementation-facing.
It applies only to `apps/control`. Managed Projects define their own tokens in
their Project Design System.

## 1. Token Rules

1. Tokens are semantic first. Component code must prefer semantic tokens over
   literal hex, pixel, or z-index values.
2. Token values are stable within a Spec Library revision.
3. A token can be deprecated only by naming the replacement and migration note.
4. The UI may not introduce local one-off colors, spacing, radii, shadows, or
   z-index values without updating this document.
5. Machine state colors must be paired with icon, Korean label, and reason
   text where applicable.

Recommended implementation location:

```text
apps/control/src/styles/tokens.css
apps/control/src/styles/theme.ts
```

## 2. Color Tokens

### 2.1 Foundation

| Token | Value | Usage |
|---|---|---|
| `--ado-color-canvas` | `#F8FAFC` | page background |
| `--ado-color-surface` | `#FFFFFF` | primary surfaces |
| `--ado-color-surface-muted` | `#F1F5F9` | secondary surfaces, inactive panels |
| `--ado-color-border` | `#E2E8F0` | default border |
| `--ado-color-border-strong` | `#CBD5E1` | emphasized boundary |
| `--ado-color-text-strong` | `#0F172A` | primary text |
| `--ado-color-text-secondary` | `#475569` | secondary text |
| `--ado-color-text-tertiary` | `#64748B` | metadata, helper text |
| `--ado-color-focus` | `#1D4ED8` | focus ring and primary action |

### 2.2 State

| Token | Value | State |
|---|---|---|
| `--ado-color-state-queued` | `#64748B` | queued, idle |
| `--ado-color-state-running` | `#0F766E` | runner-owned active work |
| `--ado-color-state-success` | `#047857` | completed, verified |
| `--ado-color-state-warning` | `#92400E` | human action, stale, conflict |
| `--ado-color-state-failure` | `#B91C1C` | failed, blocked, destructive |
| `--ado-color-state-info` | `#1D4ED8` | requested, accepted, informative |

Status presentation must use:

```text
icon + Korean label + machine value or reason when useful
```

## 3. Typography Tokens

| Token | Value |
|---|---|
| `--ado-font-sans` | `Pretendard Variable, Pretendard, Noto Sans KR, system-ui, sans-serif` |
| `--ado-font-mono` | `JetBrains Mono, ui-monospace, SFMono-Regular, Menlo, monospace` |

| Token | Size | Line height | Weight | Usage |
|---|---:|---:|---:|---|
| `--ado-text-page-title` | `24px` | `32px` | `650` | route title |
| `--ado-text-section-title` | `16px` | `24px` | `650` | panel heading |
| `--ado-text-body` | `14px` | `20px` | `400` | default UI copy |
| `--ado-text-dense` | `13px` | `18px` | `400` | tables and compact metadata |
| `--ado-text-label` | `12px` | `16px` | `600` | labels and badges |
| `--ado-text-code` | `12px` | `18px` | `450` | logs, SHA, IDs, JSON |

Korean prose uses `word-break: keep-all`. Machine identifiers use monospace and
local truncation with copy affordance.

## 4. Spacing, Radius, And Border

| Token | Value |
|---|---:|
| `--ado-space-0` | `0` |
| `--ado-space-1` | `4px` |
| `--ado-space-2` | `8px` |
| `--ado-space-3` | `12px` |
| `--ado-space-4` | `16px` |
| `--ado-space-5` | `20px` |
| `--ado-space-6` | `24px` |
| `--ado-space-8` | `32px` |
| `--ado-space-10` | `40px` |
| `--ado-space-12` | `48px` |
| `--ado-space-16` | `64px` |

| Token | Value | Usage |
|---|---:|---|
| `--ado-radius-control` | `4px` | buttons, inputs, badges |
| `--ado-radius-panel` | `6px` | panels and bounded regions |
| `--ado-radius-dialog` | `8px` | dialogs and drawers |
| `--ado-border-width` | `1px` | normal boundary |
| `--ado-focus-width` | `2px` | visible focus |

No component may use a radius above `8px` unless a future approved design
revision changes this document.

## 5. Layout Tokens

| Token | Value |
|---|---:|
| `--ado-topbar-height` | `56px` |
| `--ado-sidebar-width` | `256px` |
| `--ado-rail-width` | `64px` |
| `--ado-content-max-width` | `1600px` |

Breakpoints:

| Range | Columns | Page inset | Navigation |
|---|---:|---:|---|
| `320-479px` | 4 | `16px` | modal Drawer |
| `480-767px` | 4 | `16px` | modal Drawer |
| `768-1023px` | 6 | `20px` | icon rail |
| `1024-1439px` | 8 | `24px` | sidebar |
| `1440px+` | 12 | `32px` | sidebar |

All grid children use `min-width: 0`.

## 6. Elevation And Layering

ADO uses borders and spacing before shadow.

| Token | Value | Usage |
|---|---:|---|
| `--ado-shadow-none` | `none` | default |
| `--ado-shadow-overlay` | `0 16px 40px rgba(15, 23, 42, 0.16)` | dialogs, drawers only |

| Token | Value | Layer |
|---|---:|---|
| `--ado-z-base` | `0` | page content |
| `--ado-z-sticky` | `10` | sticky table headers, top bar |
| `--ado-z-drawer` | `40` | mobile navigation |
| `--ado-z-dialog` | `50` | modal dialogs |
| `--ado-z-toast` | `60` | transient notices |
| `--ado-z-tooltip` | `70` | tooltip and popover labels |

## 7. Motion Tokens

| Token | Value |
|---|---|
| `--ado-motion-duration-fast` | `120ms` |
| `--ado-motion-duration-base` | `160ms` |
| `--ado-motion-duration-slow` | `200ms` |
| `--ado-motion-easing-standard` | `cubic-bezier(0.2, 0, 0, 1)` |

Only opacity, color, outline, and small transform changes are allowed. Layout
motion, counter animation, infinite pulse, and infinite shimmer are prohibited.

## 8. Density Tokens

| Token | Value | Usage |
|---|---:|---|
| `--ado-control-height` | `36px` | visual height for desktop controls |
| `--ado-hit-target` | `44px` | minimum interactive target |
| `--ado-row-passive` | `40px` | passive data row |
| `--ado-row-interactive` | `44px` | clickable row |
| `--ado-panel-padding` | `16px` | default panel padding |
| `--ado-panel-padding-large` | `20px` | dense detail panel padding |

## 9. Token Verification

UI implementation PRs must prove:

1. no literal unapproved color is introduced in Control Room source;
2. spacing values come from the approved scale;
3. state colors are paired with non-color cues;
4. focus ring remains visible on keyboard paths;
5. reduced-motion mode removes nonessential transitions;
6. long machine values truncate or scroll inside bounded regions.

# Design System Reference Benchmark

This document records the external design-system references that inform ADO's
own Control Room design system and the governance rules for managed Project
design systems. It is not a visual inheritance map. ADO adopts proven operating
principles, not another product's brand.

## 1. Reference Sources

| Source | Official URL | ADO usage |
|---|---|---|
| IBM Carbon Design System | https://carbondesignsystem.com/ | Enterprise console density, component specificity, data-table discipline, design-to-code parity |
| U.S. Web Design System | https://designsystem.digital.gov/ | Accessibility-first component/pattern thinking, responsive behavior, public-sector clarity |
| Shopify Polaris | https://polaris.shopify.com/ | Admin-product content patterns, merchant-style operational clarity, tokens and component governance |
| Material Design 3 | https://m3.material.io/ | Tokenized foundation structure, state layers, motion restraint, adaptive layout concepts |
| Microsoft Fluent 2 | https://fluent2.microsoft.design/ | Cross-platform component behavior, interaction states, accessibility tooling expectations |
| W3C WCAG | https://www.w3.org/WAI/standards-guidelines/wcag/ | Minimum accessibility standard for contrast, keyboard operation, labels, and non-color cues |

References are checked for principles and structure. ADO documents must not copy
licensed component text, code, tokens, icons, illustrations, or brand assets.

## 2. Adopted Patterns

ADO adopts these common patterns across the references:

1. foundations before components: product character, tokens, typography,
   spacing, color, motion, and accessibility are defined before page specs;
2. component contracts include anatomy, states, behavior, accessibility,
   responsive behavior, and verification evidence;
3. design tokens are explicit and implementation-facing rather than embedded
   only in prose;
4. state is semantic and visible through more than color;
5. dense operational screens use restrained hierarchy, bounded scroll regions,
   and deterministic handling of long data;
6. content rules are part of the design system, especially for errors,
   destructive actions, and human approval flows;
7. accessibility is a design requirement, not a late QA pass;
8. responsive behavior is specified per component and screen, not inferred from
   generic breakpoints;
9. design changes are versioned and approved before implementation;
10. visual QA evidence is attached to the work item that changed the UI.

## 3. ADO-Specific Interpretation

ADO is not a consumer app, landing page, generic admin template, or analytics
dashboard. It is an operating console for supervising automated development.
Therefore:

- clarity beats novelty;
- committed backend state beats optimistic decoration;
- Korean human-readable labels are first-class;
- machine values keep their original English identifiers;
- every visible state must be traceable to API, DB, AuditEvent, JobAttempt, or
  EvidenceGate data;
- no UI component may imply that policy, verification, or human approval was
  bypassed.

## 4. Reference-To-ADO Mapping

| ADO area | Primary reference influence | ADO document |
|---|---|---|
| visual foundations | Carbon, Material, Fluent | `CONTROL_ROOM_DESIGN_SYSTEM.md`, `CONTROL_ROOM_DESIGN_TOKENS.md` |
| admin workflows | Polaris, Carbon | `CONTROL_ROOM_PAGE_SPECS.md`, `CONTROL_ROOM_COMPONENT_SPECS.md` |
| accessibility baseline | WCAG, USWDS | `CONTROL_ROOM_DESIGN_SYSTEM.md`, `PROJECT_DESIGN_GOVERNANCE.md` |
| tokens | Material, Polaris, Fluent | `CONTROL_ROOM_DESIGN_TOKENS.md` |
| component contracts | Carbon, USWDS, Polaris | `CONTROL_ROOM_COMPONENT_SPECS.md` |
| project-specific design systems | all references as structure only | `PROJECT_DESIGN_GOVERNANCE.md` and project templates |

## 5. Non-Adopted Patterns

ADO explicitly does not adopt:

- marketing-page hero layouts for operational screens;
- decorative gradients, blobs, or large illustration-first empty states;
- animated counters, infinite shimmer, infinite pulse, or layout-shifting motion;
- color-only status systems;
- hidden policy state;
- UI-only progress percentages that are not backed by durable evidence;
- unbounded global page scrolling for logs, JSON, tables, or long identifiers;
- automatic visual inheritance from ADO Control Room to managed Projects.

## 6. Review Requirement

Any change to Control Room design foundations, tokens, core components,
responsive matrix, accessibility promises, or project design governance must
state whether it:

1. preserves this benchmark;
2. intentionally extends it with a named reason;
3. replaces a prior rule with human approval.

The review packet must cite the affected ADO document and include UI evidence
when implementation changes are involved.

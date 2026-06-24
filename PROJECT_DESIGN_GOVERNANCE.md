# Project Design Governance

This document governs design systems for Projects managed by Agent Development
Orchestrator (ADO). It is deliberately separate from
`CONTROL_ROOM_DESIGN_SYSTEM.md`, which governs the ADO operating interface.

## 1. Two Independent Design Systems

| Layer | Owner | Applies to | Does not apply to |
|---|---|---|---|
| ADO Control Room Design System | ADO Spec Library | `apps/control`, operational consoles, approval and evidence workflows | managed Project products |
| Project Design System | one managed Project | that Project's web, mobile, desktop, game, or other UI-bearing Components | ADO Control Room |

Neither layer visually inherits from the other. Color, type, spacing, radius,
layout, component library, content voice, imagery, and motion are local to the
owning product. ADO execution and security constraints remain global, but they
are not a visual theme.

## 2. Shared Quality Baseline

Every UI-bearing Component must define, implement, and verify:

1. keyboard-operable primary paths with visible focus;
2. semantic labels, error messages, and non-color state cues;
3. stated contrast and reduced-motion behavior;
4. a responsive matrix with no unintended clipping, overlap, or page-level
   horizontal overflow;
5. loading, empty, error, permission, and recovery states where applicable;
6. deterministic handling of long text, IDs, paths, tables, and logs;
7. manual visual/interaction evidence at the approved viewport matrix.

The baseline defines outcomes, not a prescribed aesthetic. A playful game and
a dense B2B console can meet it with entirely different design systems.

## 3. Project Design System Lifecycle

For every Project with at least one UI-bearing Component:

```text
Project intake
-> inspect existing product/design evidence
-> choose provenance
-> draft Design Contract
-> define screen and responsive matrices
-> human approval
-> bind active version to planned work
-> implement and verify
-> versioned change when design direction changes
```

`provenance` is exactly one of:

- `new`: no reusable system exists; create one.
- `imported`: an external or pre-existing system is adopted without local
  visual changes.
- `extended`: an existing system is retained and documented additions are
  made.
- `not_applicable`: no user-facing UI is in scope; include a reason.

No UI implementation begins from an unapproved visual direction. If a source
product already has a design system, discovery and faithful use come before new
visual exploration.

## 4. Required Project Design Artifacts

The active ProjectConstraintProfile references immutable versions and hashes of
the applicable artifacts:

| Artifact | Purpose |
|---|---|
| `PROJECT_DESIGN_CONSTRAINTS.md` | product identity, UX principles, visual and interaction boundaries |
| `PROJECT_DESIGN_SYSTEM.md` | approved tokens, components, content, motion, and asset rules |
| `SCREEN_CATALOG.md` | every screen/state, intended user, primary question, actions, and data dependencies |
| `RESPONSIVE_MATRIX.md` | viewport breakpoints and component-level behavior at each range |
| `UI_ACCEPTANCE_CHECKLIST.md` | automated and human verification requirements per Component Work |

Only the relevant excerpts and hashes enter an Agent ContextPacket. An agent
does not receive unrelated screens, obsolete experiments, or every historical
design revision.

## 5. Human Approval And Change Control

The Human Owner approves the first active Design Contract before an UI-bearing
Feature Unit becomes executable. Approval records:

- ProjectConstraintProfile version and content hash;
- Design Contract version and content hash;
- source or visual-reference artifact IDs, when present;
- applicable Components and screen catalog version;
- viewport matrix and verification checklist version;
- known exceptions, open questions, and approval timestamp.

A Feature Unit may add a screen within the active Design Contract. It must
request a new Design Contract version when it changes a token, component rule,
visual direction, interaction pattern, responsive rule, accessibility promise,
or forbidden design behavior. Existing Component Work keeps its original bound
version until a Human Owner explicitly rebinds it with an impact note.

## 6. Implementation And Review Rules

1. A Component Work packet names the active Design Contract and exact relevant
   screens/states.
2. Implementation may not substitute the ADO Control Room aesthetic for the
   Project's approved direction.
3. Verification captures each required viewport and relevant non-happy state.
4. Review checks conformance to the Project Design System and Shared Quality
   Baseline, not personal preference or the reviewer model's default style.
5. Human verification items are emitted by Feature Unit and grouped by
   Component where useful.

## 7. Data And Audit Requirements

ADO stores the design-contract hash, screen catalog hash, responsive-matrix
hash, and checklist hash on the approved profile and effective Component Work
packet. Design approval, rebinding, and exception decisions create AuditEvents.

Raw licensed assets, private design files, and personally identifying material
are not copied into public Project repositories or browser-visible logs. Store
only authorized references, redacted metadata, and policy-permitted evidence.

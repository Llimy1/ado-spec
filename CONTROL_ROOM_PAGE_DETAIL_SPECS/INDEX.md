# Control Room Page Detail Specs

This directory contains implementation-level page detail specs for the ADO
Control Room. The canonical route list is `CONTROL_ROOM_ROUTE_REGISTRY.md`.
The parent page contracts live in `CONTROL_ROOM_PAGE_SPECS.md`.

Page detail specs are binding for:

- `apps/control` route implementation;
- `apps/api` read models and command contracts;
- `packages/contracts` DTOs;
- browser, accessibility, visual, and human verification evidence.

## 1. Required Files

| Order | File | Status |
|---:|---|---|
| 1 | `P-01_PROJECTS.md` | active |
| 2 | `P-02_PROJECT_OVERVIEW.md` | planned |
| 3 | `P-03_ROADMAPS.md` | planned |
| 4 | `P-04_FEATURE_UNIT.md` | planned |
| 5 | `P-05_COMPONENT_WORK.md` | planned |
| 6 | `P-06_RUN_DETAIL_LOGS.md` | planned |
| 7 | `P-07_VERIFICATION_REVIEW_EVIDENCE.md` | planned |
| 8 | `P-08_PR_HUMAN_VERIFICATION.md` | planned |
| 9 | `P-09_DECISIONS_INCIDENTS.md` | planned |
| 10 | `P-10_ARTIFACTS_SYSTEM_SETTINGS.md` | planned |

## 2. Detail Spec Contract

Every page detail spec must define:

1. routes and route parameters;
2. API endpoints, DTO names, and generated client functions;
3. URL state, query state, and pagination;
4. route-level component tree;
5. shared component mapping from `CONTROL_ROOM_COMPONENT_SPECS.md`;
6. loading, empty, filtered-empty, stale, disconnected, error, denied, and
   conflict states;
7. command behavior when commands exist;
8. desktop, compact desktop, tablet, and mobile layout rules;
9. keyboard, focus, live-region, and reduced-motion behavior;
10. Playwright scenarios, viewport evidence, and mock fixtures.

Detail specs must not weaken:

- `CONTROL_ROOM_API_UI_SPEC.md`
- `CONTROL_ROOM_DESIGN_SYSTEM.md`
- `CONTROL_ROOM_DESIGN_TOKENS.md`
- `CONTROL_ROOM_COMPONENT_SPECS.md`
- `CONTROL_ROOM_PAGE_SPECS.md`
- `CONTROL_ROOM_ROUTE_REGISTRY.md`

## 3. Delivery Order

A2 implementation begins with P-01 and P-02 because they form the Project read
path. A page may receive a visual shell before its backend exists only when it
shows honest unavailable or empty states and does not fabricate data.

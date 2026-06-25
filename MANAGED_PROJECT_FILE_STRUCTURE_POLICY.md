# Managed Project File Structure Policy

This document defines the default file-structure generation policy for ADO
managed product Projects.

It applies to managed Projects, not to the separate `ado-platform`
implementation repository.

## 1. Purpose

ADO must not guess where product code belongs. Before execution, every Project
has an approved `ProjectArchitecture` document that declares the repository
layout, Component roots, internal structure profile, allowed paths, generated
paths, and human-approval-only paths.

The default profile is:

```text
standard_product_monorepo
```

The profile supports products that may include web, mobile, backend API,
shared packages, design assets, infrastructure, and documentation in one
public monorepo.

## 2. Non-Negotiable Rules

1. ADO v1 manages one public monorepo per Project.
2. ADO never infers ownership from directory names alone.
3. Every Component has an explicit approved root path.
4. Every Component has an approved internal structure profile.
5. ADO agents may create or edit files only inside declared allowed paths.
6. Shared packages, contracts, migrations, lockfiles, root configuration, CI,
   and production infrastructure paths require explicit scope declaration.
7. Human-authored docs and ADO-generated docs are separated.
8. Generated files and human-edited files are separated.
9. Project-specific overrides are allowed only through approved
   `ProjectArchitecture`.

## 3. Default Root Layout

```text
project-root/
  apps/
    web/
    mobile/
    api/
  packages/
    shared/
    ui/
    contracts/
    config/
  docs/
    product/
    architecture/
    verification/
    decisions/
  assets/
    source/
    generated/
    final/
  infra/
    docker/
    deploy/
  tools/
    scripts/
  tests/
    e2e/
    fixtures/
  .ado/
    generated/
    packets/
    reports/
```

Projects may omit unused roots. For example, a backend-only service may omit
`apps/web`, `apps/mobile`, and `packages/ui`.

## 4. Root Directory Meanings

| Root | Purpose |
|---|---|
| `apps/` | runnable applications and services |
| `packages/` | reusable libraries shared by apps |
| `docs/` | human-authored product and engineering documents |
| `assets/` | source, generated, and approved final assets |
| `infra/` | non-secret infrastructure and deployment definitions |
| `tools/` | project-local scripts and development utilities |
| `tests/` | cross-component E2E tests and fixtures |
| `.ado/` | generated ADO work artifacts; not hand-edited source |

## 5. Component Root Profiles

The following Component roots are the default for
`standard_product_monorepo`.

```yaml
components:
  - key: web
    type: nextjs_web
    root: apps/web
    internal_structure_profile: nextjs_feature_app

  - key: mobile
    type: expo_mobile
    root: apps/mobile
    internal_structure_profile: expo_feature_app

  - key: api
    type: nestjs_api
    root: apps/api
    internal_structure_profile: nestjs_module_api

  - key: shared
    type: shared_library
    root: packages/shared
    internal_structure_profile: pure_shared_package

  - key: ui
    type: ui_library
    root: packages/ui
    internal_structure_profile: project_ui_package

  - key: contracts
    type: contract_library
    root: packages/contracts
    internal_structure_profile: transport_contract_package

  - key: design
    type: design_assets
    root: assets
    internal_structure_profile: design_asset_repository
```

ProjectArchitecture may rename, omit, or add Components, but it must declare
equivalent roots and structure profiles before ADO creates work.

## 6. Next.js Web Structure

Profile:

```text
nextjs_feature_app
```

Default structure:

```text
apps/web/
  src/
    app/
    features/
      {feature_key}/
        components/
        hooks/
        api/
        types.ts
    components/
    lib/
    styles/
    config/
    tests/
  public/
  package.json
```

Rules:

- `src/app` owns routing, layouts, pages, route handlers, and metadata.
- Feature implementation belongs in `src/features/{feature_key}`.
- Web-only reusable UI belongs in `src/components`.
- Cross-app UI belongs in `packages/ui`.
- API clients may live in `src/features/{feature_key}/api` or `src/lib/api`
  as declared by ProjectArchitecture.
- Tests stay near the feature or in `src/tests` when they exercise web-only
  behavior.

## 7. Mobile App Structure

Profile:

```text
expo_feature_app
```

Default structure:

```text
apps/mobile/
  src/
    app/
    screens/
    features/
      {feature_key}/
        components/
        hooks/
        api/
        types.ts
    components/
    navigation/
    lib/
    styles/
    assets/
    tests/
  app.json
  package.json
```

Rules:

- `src/app` is used when the Project selects Expo Router.
- `src/screens` is used when the Project selects explicit navigation stacks.
- Feature implementation belongs in `src/features/{feature_key}`.
- Server communication belongs in feature-local `api` modules or an approved
  mobile API client module.
- Platform-specific files use explicit suffixes such as `.ios.tsx` and
  `.android.tsx`.
- Shared business types belong in `packages/contracts` or `packages/shared`,
  not duplicated inside mobile code.

## 8. Backend API Structure

Profile:

```text
nestjs_module_api
```

Default structure:

```text
apps/api/
  src/
    main.ts
    app.module.ts
    modules/
      {module_key}/
        {module_key}.module.ts
        {module_key}.controller.ts
        {module_key}.service.ts
        {module_key}.repository.ts
        dto/
        entities/
        policies/
        tests/
    common/
      guards/
      filters/
      pipes/
      interceptors/
    config/
    database/
      migrations/
      seeds/
    health/
  test/
  package.json
```

Rules:

- `modules/{module_key}` is the default backend feature boundary.
- Controllers stay at the transport boundary.
- Services/use cases own mutations.
- Repositories/query services own persistence reads and writes.
- DTOs are transport types; DB entities are not exported as API contracts.
- Migrations are reviewed files and are never generated as hidden side
  effects of an agent run.
- Shared transport contracts belong in `packages/contracts`.

## 9. Shared Package Structure

Profile:

```text
pure_shared_package
```

Default structure:

```text
packages/shared/
  src/
    domain/
    utils/
    validation/
    constants/
    types/
    index.ts
  tests/
  package.json
```

Rules:

- No React, Next.js, NestJS, Expo, or database imports.
- Only pure domain helpers, constants, validation, and shared types are
  allowed.
- If a helper needs framework context, it belongs in the consuming app or a
  dedicated package approved by ProjectArchitecture.

## 10. UI Package Structure

Profile:

```text
project_ui_package
```

Default structure:

```text
packages/ui/
  src/
    components/
    primitives/
    tokens/
    hooks/
    styles/
    index.ts
  tests/
  package.json
```

Rules:

- This package implements the managed Project design system.
- It does not inherit the ADO Control Room design system.
- Primitives and tokens are framework choices approved in
  ProjectArchitecture.
- App-specific screens do not live here.

## 11. Contracts Package Structure

Profile:

```text
transport_contract_package
```

Default structure:

```text
packages/contracts/
  src/
    api/
    events/
    schemas/
    index.ts
  tests/
  package.json
```

Rules:

- Transport contracts shared by frontend, mobile, and backend live here.
- DB entity types are not exported as transport contracts.
- Schema technology such as OpenAPI, Zod, TypeBox, or JSON Schema is selected
  per ProjectArchitecture.
- Public event names and payloads are versioned here when used by multiple
  Components.

## 12. Config Package Structure

Profile:

```text
project_config_package
```

Default structure:

```text
packages/config/
  src/
    env/
    constants/
    feature-flags/
    index.ts
  tests/
  package.json
```

Rules:

- Shared non-secret configuration helpers may live here.
- Raw secrets and raw `.env` files never live here.
- Runtime-specific secret loading stays in the app or deployment layer.

## 13. Assets Structure

Profile:

```text
design_asset_repository
```

Default structure:

```text
assets/
  source/
  generated/
  final/
```

Rules:

- `source` stores human-provided source assets.
- `generated` stores tool or AI generated candidates.
- `final` stores human-approved final assets.
- Public repositories must not contain private source material, PII, or
  license-unsafe assets.

## 14. ADO Generated Files

ADO-generated files stay under:

```text
.ado/
  generated/
  packets/
  reports/
```

Rules:

- `.ado/generated` contains generated project specs, Feature Unit specs,
  Component Work specs, and review summaries when the Project chooses
  repository-local artifacts.
- `.ado/packets` contains local copies of generated agent packets when
  allowed by policy.
- `.ado/reports` contains verification summaries safe for repository storage.
- Raw logs, raw model outputs, private packets, and unredacted artifacts do
  not belong in the Project repository.

## 15. Human-Authored Docs

Human-authored project docs use:

```text
docs/
  product/
  architecture/
  verification/
  decisions/
```

Rules:

- Product intent and user-facing decisions belong in `docs/product`.
- Architecture decisions and approved overrides belong in
  `docs/architecture`.
- Human verification checklists and project-level verification guides belong
  in `docs/verification`.
- Decision records belong in `docs/decisions`.
- Docs do not become ADO source of truth unless imported, validated, and
  linked to DB records.

## 16. Test Placement

Default test placement:

```text
apps/web/src/**/*.test.tsx
apps/mobile/src/**/*.test.tsx
apps/api/src/**/*.spec.ts
packages/shared/src/**/*.test.ts
packages/ui/src/**/*.test.tsx
packages/contracts/src/**/*.test.ts
tests/e2e/
tests/fixtures/
```

Rules:

- Unit tests live near the code they verify.
- Component-specific integration tests live in that Component root.
- Cross-component E2E tests live in `tests/e2e`.
- Fixtures must not contain secrets, PII, production data, or private packets.

## 17. Default Allowed Path Profiles

ADO derives initial AllowedPathRule proposals from the approved Component
roots and structure profiles. Human approval is still required before
execution.

### Web Work

```text
apps/web/**
packages/ui/**
packages/shared/**
packages/contracts/**
docs/verification/**
```

### Mobile Work

```text
apps/mobile/**
packages/ui/**
packages/shared/**
packages/contracts/**
docs/verification/**
```

### API Work

```text
apps/api/**
packages/shared/**
packages/contracts/**
docs/verification/**
```

### Shared Library Work

```text
packages/shared/**
packages/contracts/**
docs/verification/**
```

### UI Library Work

```text
packages/ui/**
assets/final/**
docs/architecture/**
docs/verification/**
```

### Design Asset Work

```text
assets/source/**
assets/generated/**
assets/final/**
docs/product/**
docs/decisions/**
```

Shared path edits require coordinated Component Work when they affect more
than one Component's acceptance or verification.

## 18. Default Forbidden Paths

ADO agents must not edit these paths without an explicit HumanDecision and
Component Work scope:

```text
.env
.env.*
**/secrets/**
**/credentials/**
**/*.pem
**/*.key
.github/workflows/**
infra/deploy/production/**
```

Even with approval, secrets and raw credential values must not be committed.

## 19. Project-Specific Overrides

Projects may define profiles such as:

```text
web_only_product
mobile_api_product
fullstack_product
backend_only_service
design_asset_project
```

An override must state:

- selected layout profile;
- Component keys, types, roots, and internal structure profiles;
- omitted default roots;
- added custom roots;
- generated paths;
- forbidden paths;
- human approval required paths;
- verification commands per Component.

ADO treats an undefined root as unavailable. It does not create directories
outside the approved ProjectArchitecture.

## 20. Acceptance Criteria

This policy is satisfied for a Project when:

1. ProjectArchitecture declares a layout profile.
2. Every Component has a root path and internal structure profile.
3. Every generated path is distinct from human-authored source paths.
4. Allowed paths can be derived without guessing.
5. Forbidden paths and human-approval-only paths are explicit.
6. Verification commands know where unit, integration, E2E, and fixture files
   belong.
7. ADO can explain why every created or modified file belongs to the approved
   Component Work scope.

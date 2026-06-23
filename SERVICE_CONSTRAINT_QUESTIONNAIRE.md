# Service Constraint Questionnaire

This questionnaire is used to create a project-specific constraint profile.

It should be answered once when creating a new Project and updated when the service direction changes.

## 1. Product Identity

- What is the product or service name?
- What problem does it solve?
- Who is the primary user?
- Who is the secondary user?
- What should this product never become?
- What products or experiences are useful references?
- What references should be avoided?

## 2. Scope

- What components exist now?
- What components are planned?
- Which repositories map to each component?
- Which platforms are in scope?
- Which platforms are out of scope?
- Is this a new product, existing product, rewrite, or extension?

## 3. UX And Product Tone

- What should the product feel like?
- Should it be playful, professional, premium, calm, dense, minimal, expressive, or operational?
- What are the top three UX principles?
- What user actions must be fast?
- What user actions require confirmation?
- What empty states matter?
- What error states matter?

## 4. Visual Design

- Is there an existing design system?
- What colors, typography, spacing, and layout rules already exist?
- Are there brand assets?
- Are generated images allowed?
- Are stock-like images allowed?
- Are illustrations acceptable?
- What visual styles are forbidden?

## 5. Frontend Architecture

- Which frontend frameworks are used?
- Which routing system is used?
- Which state management pattern is used?
- Which component library is used?
- What accessibility level is expected?
- What browsers or devices must be supported?
- What frontend testing commands exist?

## 6. Backend Architecture

- Which backend frameworks are used?
- Which API style is used?
- Which database is used?
- Which auth system is used?
- Which background job system is used?
- Which external services are used?
- What backend testing commands exist?

## 7. Mobile Architecture

- Which mobile framework is used?
- Which platforms are supported?
- Which devices or simulators must be tested?
- Which native permissions are used?
- Which store or build constraints matter?
- What mobile testing commands exist?

## 8. Data And Privacy

- What data is sensitive?
- Is PII stored?
- Are production data samples allowed in development?
- What data must never leave the local machine?
- What retention rules matter?
- What audit requirements matter?

## 9. Verification

- What must be automatically tested?
- What must be manually verified by a human?
- Which commands prove success?
- Which screenshots, logs, or recordings are useful evidence?
- Which flows are critical?
- Which performance thresholds matter?
- Which accessibility checks matter?

## 10. Release And Git

- Which branch is protected?
- Which branch receives ADO PRs?
- What branch naming convention should be used?
- Are draft PRs preferred?
- Who reviews and merges?
- What CI checks must pass before human merge?

## 11. Constraints And Forbidden Behavior

- What files or directories must agents never edit?
- What commands must agents never run?
- What design choices are forbidden?
- What architecture choices are forbidden?
- What dependency changes need human approval?
- What external network calls need human approval?

## 12. Budget And Model Use

- Which paid tools are allowed?
- Which paid tools are forbidden?
- Are local models preferred for review?
- What token or cost limits matter?
- When should work stop and ask the human?

## 13. Human Verification Checklist Preferences

- Should checklists be grouped by component?
- Should screenshots be required?
- Should API request/response samples be required?
- Should mobile device checks be required?
- Should accessibility checks be required?
- Should rollback notes be required?

## 14. Output

ADO uses the answers to generate:

- `PROJECT_SPEC.md`
- `PROJECT_DESIGN_CONSTRAINTS.md`
- `PROJECT_ARCHITECTURE.md`
- `PROJECT_VERIFICATION_PROFILE.md`
- project-specific Feature Unit defaults
- project-specific Component Work defaults

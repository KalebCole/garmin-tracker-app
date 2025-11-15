<!--
SYNC IMPACT REPORT
Version change: 1.0.0 → 1.1.0
Modified principles:
  - IV. Simple State Machines, Zero Ambiguity (softened state diagram requirement)
Added principles:
  - VI. Velocity First for v1
Added sections:
  - Technical Foundations
  - Development Workflow
Removed sections:
  - None
Templates checked and status:
  - ✅ .specify/templates/plan-template.md
  - ✅ .specify/templates/spec-template.md
  - ✅ .specify/templates/tasks-template.md
  - ⚠ .specify/templates/commands/*.md (not inspected) — verify agent-specific references
  - ⚠ README.md (not present) — manual sync may be required
  - ⚠ docs/quickstart.md (not present) — manual sync may be required
Follow-up TODOs:
  - TODO(PROJECT_NAME): Confirm project display name ("Garmin Tracker") if different.
  - TODO: Review .specify/templates/commands/*.md and public docs for agent-specific language.
-->

## Core Principles

### I. Garmin-First, Fast-Log Experience
- Principle: User interactions on the Garmin device MUST prioritize minimal latency and minimal steps.
- Rules:
  - Input flows MUST complete in ≤ 4 direct interactions for primary log types
  - No feature MAY introduce additional mandatory screens that slow logging
  - UI choices MUST favor button navigation where touchscreen is unavailable
- Rationale:
  - The core product value is fast, reliable on-device logging. Any change that
    degrades speed or adds friction undermines user utility and is disallowed
    without explicit amendment.

### II. One Data Model Across All Surfaces
- Principle: A single canonical schema MUST be shared by watch, backend, and mobile.
- Rules:
  - Trackable entities MUST include: type (DURATION | COUNT | WEIGHT),
    value format, iconId
  - Entry records MUST include an ISO-8601 timestamp and a single numeric value
  - Any schema change that affects stored fields MUST include a migration plan
- Rationale:
  - A unified schema prevents costly refactors when adding sync or analytics
    and ensures interoperability across devices and services.

### III. Local-First, Offline-Guaranteed
- Principle: The watch app MUST function fully offline for v1 functionality.
- Rules:
  - All primary features (view trackables, create entries, confirm/save/discard)
    MUST work without network access
  - Sync (if implemented) MUST be additive and never required for core flows
- Rationale:
  - Many target devices and scenarios have intermittent network; local-first
    ensures reliability and predictable user experience.

### IV. Simple State Machines, Zero Ambiguity
- Principle: Every user flow MUST map to a concise deterministic state machine.
- Rules:
  - All flows MUST be documented as state diagrams before implementation
  - No hidden shortcuts or nondeterministic transitions are allowed
  - Inputs and button mappings MUST be explicit and testable
- Rationale:
  - Deterministic state machines reduce bugs on constrained devices and make
    testing and verification straightforward.
### V. Minimal Surface Area (Simplicity Gate)
- Principle: Only features that directly support the three MVP log types are
  allowed in v1.
- Rules:
  - Features outside duration, count, or weight logging are disallowed for v1
  - Any proposed expansion MUST include evidence it preserves logging speed,
    state-machine clarity, and schema simplicity
- Rationale:
  - YAGNI discipline prevents feature creep that would harm core usage goals.

### VI. Velocity First for v1
- Principle: Documentation and testing overhead MUST be proportionate to v1 scope.
- Rules:
  - Documentation MUST NOT exceed the complexity of the feature it protects
  - Tests for v1 MUST be smoke tests only
  - Diagrams required ONLY for multi-step flows that involve 3+ screens
- Rationale:
  - Over-engineering docs/tests for a simple v1 MVP slows delivery and creates
    maintenance burden that outweighs benefits. We optimize for shipping the core
    product quickly.

## Technical Foundations
- Device target: fr945lte (button navigation, 240×240 round display)
- Rendering constraints: max 1 Hz UI updates; high-contrast, centered elements
- Storage: Use Storage or Application.Properties for entries; hardcoded trackables
  for v1
- Backend (future): Postgres DB; TypeScript backend; endpoint contracts
  pre-defined for later sync work

## Development Workflow
- Phase-driven delivery:
  1. Garmin local-only MVP (Carousel, Duration, Count, Weight, Confirm)
  2. Optional backend + sync (push-only)
  3. Optional mobile app (trackable management, viewer)
- State-machine-first: Every view begins as a written state diagram
- Simplicity Gate (Non-negotiable): Features that increase surface area beyond
  stated limits must be deferred unless explicitly ratified

## Governance
- Authority: This constitution supersedes informal preferences; it guides
  design and review decisions.
- Amendment procedure:
  1. Submit an amendment PR that includes: concrete use case, performance
     impact analysis, state-machine diagrams, and a migration plan if schema
     changes are proposed.
  2. Minimum approval: 2 maintainers with at least one reviewer who authored a
     prior accepted amendment.
  3. Amendments that change or remove principles are a MAJOR change (see
     Versioning policy below) and require explicit justification plus a rollout
     plan.
- Versioning policy:
  - MAJOR: Backward-incompatible changes (removing/renaming principles,
    schema-breaking changes)
  - MINOR: Adding principles or materially expanding guidance
  - PATCH: Clarifications, wording improvements, typo fixes
  - Initial release set to 1.0.0
- Compliance review expectations:
  - All feature plans (plan.md) MUST include a "Constitution Check" with
    explicit pass/fail for affected principles
  - PRs that touch user flows or data schemas MUST reference the relevant
    principle checks in the changelog or PR description
  - Each feature PR must state which core principle(s) it affects and whether it violates any. No formal review ceremony required.
**Version**: 1.1.0 | **Ratified**: 2025-11-15 | **Last Amended**: 2025-11-15

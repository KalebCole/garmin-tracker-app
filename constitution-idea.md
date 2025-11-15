racker Constitution (v1)
Core Principles
I. Garmin-First, Fast-Log Experience

Every design choice prioritizes speed of logging on the Garmin watch:

Open → pick trackable → log → save/discard → done.

Minimal screens, minimal friction, minimal scrolling.

Inputs must be fast even with no touchscreen (buttons only).

II. One Data Model Across All Surfaces

The watch app, backend, and mobile app must share one universal schema:

Trackable defines type (DURATION / COUNT / WEIGHT), value format, icon.

Entry stores timestamp + numeric value.

This ensures that future syncing, mobile UIs, or analytics require no refactoring.

III. Local-First, Offline-Guaranteed

The watch must fully function offline, with:

Local storage of trackables (hardcoded for v1).

Local storage of entries in a persistent store.

No network assumptions.

Future sync must be additive, not required.

IV. Simple State Machines, Zero Ambiguity

All Garmin flows must map directly onto clean, deterministic state machines:

HOME → LOG → CONFIRM → HOME

HOME → LOG → STATS → HOME

No long presses (unless proven available on this model).

No hidden shortcuts; every path is fully deterministic.

V. Minimal Surface Area (YAGNI for Real)

If it does not serve the three MVP log types (duration, count, weight), it is out:

No charts, reminders, auto-detection, streaks, achievements, sensors.

No on-watch CRUD for trackables.

No time windows beyond “all-time” and “last value” for weight.

Future expansion is allowed only if it does not compromise:

Logging speed

Clarity of state machines

Simplicity of data model

Technical Foundations
Garmin Application Constraints

App type: watch-app.

Device target: fr945lte (button navigation, 240×240 round display).

Rendering:

Max 1 Hz updates.

High-contrast UI with centered elements.

Storage:

Must use Storage or Application.Properties for entries.

Hardcoded trackables for v1.

No long-press reliance unless verified to be natively supported.

Backend / API (Future)

Even though v1 is local-only, the backend contract is pre-defined:

DB: Postgres.

Backend: TypeScript.

Tables:

trackables

entries

Endpoints (future):

GET/POST trackables

GET/POST entries

Icons

Icons referenced by iconId

Watch maps these to monochrome bitmaps preloaded in resources.

Mobile/backend may store SVG versions for richer display.

Development Workflow
1. Build in Three Phases

Garmin Local-Only MVP (must fully work without backend):

Carousel

Duration Log

Count Log

Weight Log

Save/Discard/Cancel screen

Stats View (local entries only)

Backend + Sync (optional future milestone):

Implement API contracts

Implement “push-only” sync from watch to backend

Mobile App (optional future milestone):

Trackable management

Entry viewer

Simple stats

2. State-Machine-First Implementation

Every view starts with a written state diagram before coding:

HOME

LOG_DURATION

LOG_COUNT

LOG_WEIGHT

CONFIRM

STATS

Buttons → transitions → outcomes must be unambiguous.

3. Simplicity Gate (Non-Negotiable)

Any feature requiring:

More than 1 additional view

More than 1 new model field

A nontrivial navigation update

must be deferred unless it directly supports duration, count, or weight logging.

Governance

This constitution overrides any future refactoring pressure or feature sprawl.

Amendments require:

A concrete use case

Demonstration that it does not degrade logging speed

Confirmation that state machine clarity is preserved

Any PR or code decision must follow:

Would this slow down logging?

Would this complicate the state machine?

Would this change the shared data model?
If yes → reject or redesign.

Version: 1.0.0
Ratified: 2025-11-15
Last Amended: 2025-11-15
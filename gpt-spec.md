Yep, we can absolutely support “decimal vs integer” at the routine level, and your instincts on the save/discard/cancel + custom icons are all solid.

Let’s lock this into a clear spec so you can build without second-guessing yourself.

---

# Tracker – Garmin Watch App & Backend Spec (v1)

## tl;dr

A **Garmin Forerunner 945 LTE watch app** that lets you quickly log:

* **Durations** (e.g., cold plunge)
* **Counts** (e.g., pushups)
* **Weights** (e.g., body weight)

using a **circular carousel** of routines (trackables). Each entry is:

* linked to a **trackable**
* has a **timestamp**
* has a **numeric value** (seconds, reps, or weight)

v1:

* **Garmin-first**, with **local storage only**
* No charts, social, reminders, auto-detection
* A future **TypeScript + Postgres backend** will mirror the same data model

---

## Problem Statement

You want a way to log fitness-related actions on your **Garmin watch** with **minimal phone usage** and **minimal friction**:

* Existing tools are too heavy (full workout apps or habit trackers).
* You often just want to record a *quick*, *simple* value: time in cold plunge, reps, or body weight.
* You’d like these logs to be structured, timestamped, and eventually accessible in a database.

---

## Goals

### Business / Personal Goals

* Have a **single Garmin watch app** that can log multiple custom fitness metrics.
* Make logging **fast enough** that it doesn’t feel like “using your phone on your wrist.”
* Use a **data model** that can later be synced to a **TypeScript + Postgres** backend without major refactors.

### User Goals (You / Power-user)

* Open the app → **select routine → log → done** in seconds.
* Support three core trackable types:

  * **DURATION** (cold plunge)
  * **COUNT** (pushups)
  * **WEIGHT** (body weight, decimal)
* See simple **stats** on the watch:

  * All-time total for DURATION / COUNT
  * Last logged value for WEIGHT

### Non-Goals (v1)

* No social/sharing.
* No notifications/reminders.
* No auto-detection (e.g., sensors).
* No charts/graphs on the watch.
* No complex filtering or analytics.
* No on-watch creation/edit/deletion of trackables (that will be done via mobile/backend later).

---

## Trackable Types & Value Formats

Yes — we make **value formatting** explicit at definition time.

### Trackable Types

```ts
type TrackableType = "DURATION" | "COUNT" | "WEIGHT";
```

* `DURATION`: Stored in **seconds**
* `COUNT`: Stored as **integer reps**
* `WEIGHT`: Stored as **decimal** (e.g., pounds or kg)

### Value Format

Optional extra field to explicitly control integer vs decimal & step size:

```ts
type ValueFormat = "INT" | "DECIMAL_1";

interface Trackable {
  id: string;
  name: string;
  type: TrackableType;
  valueFormat: ValueFormat;    // COUNT → INT, WEIGHT → DECIMAL_1
  iconId: string;              // maps to your custom SVG/icon asset
}
```

* For **COUNT**: `valueFormat = "INT"`; step `+/-1` or `+/-5`
* For **WEIGHT**: `valueFormat = "DECIMAL_1"`; effectively step `+/-0.1`

On the **watch**, you’ll map `iconId` → bitmap/graphic (not actual SVG, but logically “icon key”) in resources.

---

## Data Model

Even though v1 is local-only on the watch, we design it to match a future Postgres backend.

### Trackable

```ts
type TrackableType = "DURATION" | "COUNT" | "WEIGHT";
type ValueFormat = "INT" | "DECIMAL_1";

interface Trackable {
  id: string;          // "cold_plunge", "pushups", "weight"
  name: string;        // "Cold Plunge"
  type: TrackableType;
  valueFormat: ValueFormat;
  iconId: string;      // e.g. "icon_cold_plunge"
}
```

### Entry

```ts
type EntrySource = "GARMIN" | "MOBILE";

interface Entry {
  id: string;          // uuid-like string
  trackableId: string;
  timestamp: string;   // ISO string (UTC)
  value: number;       // seconds, reps, or weight
  source: EntrySource; // "GARMIN" for v1
}
```

### Local Storage (Garmin)

* `trackables`: stored as JSON (hardcoded for v1, later fetched/synced)
* `entries`: stored as an array of `Entry` in `Storage` or `Application.Properties`

Later, backend sync will just POST these entries as-is.

---

## User Stories

1. **As a user**, I can choose a routine (Cold Plunge, Pushups, Weight) from a **carousel** so I can quickly log different metrics.
2. **As a user**, I can log a **DURATION** by starting/stopping a timer and then choosing whether to save or discard.
3. **As a user**, I can log a **COUNT** with integer input and confirm via Save/Discard/Cancel.
4. **As a user**, I can log my **WEIGHT** with decimal input and confirm via Save/Discard/Cancel.
5. **As a user**, I can see a simple **Stats View**:

   * For DURATION/COUNT: **all-time total**.
   * For WEIGHT: **last logged value**.
6. **As a user**, I can navigate back out of any screen using the BACK button in a way that feels predictable and consistent.

---

## Garmin UX – Screens & Controls

### Device buttons (recap)

* **UP**: navigate/scroll up
* **DOWN**: navigate/scroll down
* **START**: confirm / start / stop
* **BACK**: cancel / exit / back

No long-press behavior is assumed (we design only with normal presses).

---

### Screen 1 – Home Carousel (Trackable Picker)

**Purpose:** choose which routine to log or view stats for.

**Layout (240×240 round):**

* Center: large icon for current trackable (`iconId` mapped to bitmap)
* Below center: trackable `name`
* Optional top text: `"Tracker"` or date
* Optional bottom hint text: `UP/DOWN: Select • START: Log`

**State:**

* `trackables: Trackable[]`
* `currentIndex: Int`

**Controls:**

* `UP`:

  * `currentIndex = (currentIndex - 1 + trackables.length) % trackables.length`
  * `requestUpdate()`
* `DOWN`:

  * `currentIndex = (currentIndex + 1) % trackables.length`
  * `requestUpdate()`
* `START`:

  * Open **Log View** for `trackables[currentIndex]`
* `BACK`:

  * Exit app

Stats are accessed via Log View (next section), not directly from Home.

---

### Screen 2 – Log View (Per Trackable)

Behavior depends on `trackable.type`:

* `DURATION` → Timer subgroup
* `COUNT` → Integer numeric subgroup
* `WEIGHT` → Decimal numeric subgroup

#### 2.1 Duration Log (Cold Plunge)

**States:**

* `IDLE` (timer 00:00, not running)
* `RUNNING` (timer counting)
* `DONE` (handled via Save/Discard/Cancel screen)

**Layout:**

* Top: `"Cold Plunge"`
* Center: big `MM:SS` or `HH:MM:SS`
* Bottom hints:

  * IDLE: `START: Start • BACK: Stats`
  * RUNNING: `START: Stop • BACK: Cancel`

**Controls:**

* In `IDLE`:

  * `START` → set `startTime = Time.now()`, `isRunning = true`, state = `RUNNING`
  * `BACK` → open **Stats View** for this trackable
* In `RUNNING`:

  * `START`:

    * Compute `elapsedSeconds = Time.now() - startTime`
    * Transition to **Save/Discard/Cancel** with `value = elapsedSeconds`
  * `BACK`:

    * Cancel current session
    * Return to **Home Carousel**

A 1 Hz `Timer` recomputes `elapsedSeconds` during `RUNNING` and calls `requestUpdate()`.

> Note: Stats are intentionally not reachable while the timer is running to keep behavior simple.

---

#### 2.2 Count Log (Pushups)

**State:**

* `count: Int` (initial default, e.g. 10)
* No separate “running” state.

**Layout:**

* Top: `"Pushups"`
* Center: large number `count`
* Bottom hints:
  `UP/DOWN: Adjust • START: Save • BACK: Stats`

**Controls:**

* `UP`:

  * `count += 1` (or `+5` if you prefer)
  * `requestUpdate()`
* `DOWN`:

  * `count = max(0, count - 1)`
  * `requestUpdate()`
* `START`:

  * Go to **Save/Discard/Cancel** with `value = count`
* `BACK`:

  * Open **Stats View** for this trackable

---

#### 2.3 Weight Log (Body Weight)

Same structure as Count, different formatting:

**State:**

* `weight: Number` (e.g. `175.0`)

**Layout:**

* Top: `"Weight"`
* Center: large number formatted with 1 decimal (`175.8`)
* Bottom hints:
  `UP/DOWN: Adjust • START: Save • BACK: Stats`

**Controls:**

* `UP`:

  * `weight += 0.1`
* `DOWN`:

  * `weight -= 0.1` but clamp to a sane min (e.g. `>= 0`)
* `START`:

  * Go to **Save/Discard/Cancel** with `value = weight`
* `BACK`:

  * Open **Stats View** for this trackable

Formatting is controlled by `valueFormat = "DECIMAL_1"`.

---

### Screen 3 – Save / Discard / Cancel (Confirmation)

**Purpose:** avoid accidental logs and mimic Garmin run flow.

**Layout:**

* Top: trackable name
* Middle: value you’re saving:

  * For DURATION: `MM:SS` or `HH:MM:SS`
  * For COUNT: integer reps
  * For WEIGHT: decimal with 1 decimal place
* Bottom: vertically stacked options, one selected:

  ```
  > Save
    Discard
    Cancel
  ```

**State:**

* `value: number`
* `selectionIndex: 0 | 1 | 2`  // 0 = Save, 1 = Discard, 2 = Cancel

**Controls:**

* `UP`:

  * `selectionIndex = max(0, selectionIndex - 1)`
* `DOWN`:

  * `selectionIndex = min(2, selectionIndex + 1)`
* `START`:

  * If `Save`:

    * Create `Entry`:

      * `trackableId`
      * `timestamp = Time.now()`
      * `value`
      * `source = "GARMIN"`
    * Persist locally
    * Haptic vibrate
    * Pop back to **Home Carousel**
  * If `Discard`:

    * Do not save
    * Pop back to **Home Carousel**
  * If `Cancel`:

    * Return to previous **Log View** with the value still visible
* `BACK`:

  * Same as selecting **Cancel** (return to Log View)

---

### Screen 4 – Stats View (Per Trackable)

**Purpose:** glanceable stats with minimal complexity.

**Stats rules:**

* `DURATION`:

  * `All-time total duration`
* `COUNT`:

  * `All-time total count`
* `WEIGHT`:

  * `Last logged weight`

**Layout:**

* Top: trackable name
* Center:

  * For DURATION:
    `All-time: 3h 12m` (computed from sum of `value` seconds across entries)
  * For COUNT:
    `All-time: 4,280 reps`
  * For WEIGHT:
    `Last: 175.8 lbs`
* Bottom hint:
  `START: Log • BACK: Home`

**Controls:**

* `START`:

  * Go to **Log View** for this trackable
* `BACK`:

  * Go back to **Home Carousel`
* `UP/DOWN`:

  * No behavior in v1 (can be used later for more stats windows)

**Navigation Summary:**

* From **Home**: `START` → **Log**
* From **Log**: `BACK` → **Stats**
* From **Stats**:

  * `START` → **Log**
  * `BACK` → **Home**

This avoids long presses entirely and keeps a simple mental model:

* Home ↔ Log ↔ Stats are a small triangle.

---

## Technical Considerations

### Garmin App

* App type: `watch-app` targeting `fr945lte`.
* Use given skeleton:

  * `MyApp` extends `App.AppBase`
  * `MyView`/`MyDelegate` derived from `WatchUi.View` and `WatchUi.BehaviorDelegate`.
* Screen updates:

  * `onUpdate(dc)` draws per current **screen state**.
  * Use a simple **finite state machine** with enums: `HOME`, `LOG_DURATION`, `LOG_COUNT`, `LOG_WEIGHT`, `CONFIRM`, `STATS`.
* Timer:

  * Use a 1 Hz `Timer.Timer` only when:

    * In `LOG_DURATION` and `isRunning = true`.
* Storage:

  * Use `Storage` or `Application.Properties` to persist:

    * `trackables` (hardcoded for now; optional)
    * `entries` array
* Icons:

  * `iconId` → map to resource bitmap IDs in `resources.xml`.
  * On the watch, you don’t deal with SVG; you just reference named bitmaps.

### Backend (Future – TypeScript + Postgres)

We won’t build this v1, but the schema should match the watch.

#### Tables

```sql
CREATE TABLE trackables (
  id           TEXT PRIMARY KEY,
  name         TEXT NOT NULL,
  type         TEXT NOT NULL CHECK (type IN ('DURATION', 'COUNT', 'WEIGHT')),
  value_format TEXT NOT NULL CHECK (value_format IN ('INT', 'DECIMAL_1')),
  icon_id      TEXT NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE entries (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  trackable_id  TEXT NOT NULL REFERENCES trackables(id) ON DELETE CASCADE,
  timestamp     TIMESTAMPTZ NOT NULL,
  value         NUMERIC NOT NULL,
  source        TEXT NOT NULL CHECK (source IN ('GARMIN', 'MOBILE')),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### TS interfaces

```ts
type TrackableType = "DURATION" | "COUNT" | "WEIGHT";
type ValueFormat = "INT" | "DECIMAL_1";
type EntrySource = "GARMIN" | "MOBILE";

interface TrackableRow {
  id: string;
  name: string;
  type: TrackableType;
  value_format: ValueFormat;
  icon_id: string;
  created_at: string;
}

interface EntryRow {
  id: string;
  trackable_id: string;
  timestamp: string;
  value: number;
  source: EntrySource;
  created_at: string;
}
```

---

## Milestones & Sequencing

### Milestone 1 – Garmin Local-Only MVP (XX weeks)

* Hardcode 3 trackables in code:

  * `cold_plunge` (DURATION, INT seconds)
  * `pushups` (COUNT, INT)
  * `weight` (WEIGHT, DECIMAL_1)
* Implement:

  * Home Carousel
  * Duration Log (with timer)
  * Count Log
  * Weight Log
  * Confirmation screen (Save/Discard/Cancel)
  * Stats View (using locally stored entries)
* Persist entries between runs with `Storage`.

### Milestone 2 – TypeScript + Postgres Backend (XX weeks)

* Create Postgres DB with `trackables` and `entries`.
* Implement minimal REST API:

  * `GET /trackables`
  * `POST /trackables`
  * `GET /entries?trackableId=&from=&to=`
  * `POST /entries`
* Add simple scripts or a small web UI to view data.

### Milestone 3 – Mobile App (Expo) (XX weeks)

* Expo app:

  * List trackables
  * Add/edit/delete trackables
  * View recent entries per trackable
  * View same stats as watch or richer ones

---

If you want next, I can:

* Write out the **state machine** for the Garmin app in more code-like pseudocode (`onKey` switching), or
* Help you define the **exact hardcoded trackable list & initial values** you’ll ship in v1.

Which do you want to tackle first: the Garmin state machine or the initial code-level definitions for `Trackable` and `Entry` on the watch?

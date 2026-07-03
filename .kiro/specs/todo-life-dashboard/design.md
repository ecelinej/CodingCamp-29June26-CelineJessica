# Design Document: To-Do Life Dashboard

## Overview

The To-Do Life Dashboard is a self-contained single-page web application delivered as one HTML file with embedded CSS and JavaScript — no build tooling, no frameworks, no external dependencies. It runs entirely in the browser and persists all user data through the Web Storage API (`localStorage`).

The application is composed of five independent UI widgets arranged on a single page:

1. **Greeting Widget** — time, date, contextual greeting, customizable name
2. **Focus Timer** — 25-minute Pomodoro countdown with start/stop/reset
3. **To-Do List** — full CRUD for tasks with completion toggling and sorting
4. **Quick Links** — add/delete labelled URL shortcuts
5. **Theme Toggle** — light/dark mode with persistence

All logic is pure vanilla JavaScript (ES6+). There are no module bundlers, transpilers, or package managers involved.

---

## Architecture

### Single-File Structure

The entire application lives in one file: `index.html`. The document is organized into three logical zones:

```
index.html
├── <head>           — meta, title, embedded <style> (CSS custom properties + widget styles)
├── <body>           — semantic HTML markup for all widgets
└── <script>         — all application logic (deferred, runs after DOM is ready)
```

### Module Pattern (IIFE per Widget)

Since no module system is available, each widget's logic is wrapped in an **Immediately Invoked Function Expression (IIFE)** to avoid polluting the global scope. A thin shared namespace (`window.App`) is used for cross-widget communication (e.g., theme changes).

```
window.App = {}           ← shared namespace
App.Storage               ← localStorage abstraction layer
App.GreetingWidget        ← greeting + clock logic
App.FocusTimer            ← countdown timer logic (with custom duration)
App.TodoList              ← task list CRUD + sorting + persistence + duplicate check
App.Stats                 ← productivity statistics tracking + persistence
App.QuickLinks            ← link CRUD + persistence
App.ThemeManager          ← theme toggle + persistence
```

### Data Flow

```
User Interaction
      │
      ▼
Widget Handler (validates input)
      │
      ├─ Update in-memory state (array/object)
      │
      ├─ Persist to localStorage via App.Storage
      │
      └─ Re-render DOM from state
```

State is always kept in memory as plain JS objects/arrays. The DOM is treated as a pure view — it is rebuilt from state on every mutation (re-render approach, no two-way binding).

### Event Strategy

All user events are registered once on DOMContentLoaded using **event delegation** where practical (e.g., task list buttons delegated to the list container). Clock ticks use `setInterval`.

---

## Components and Interfaces

### App.Storage

A thin wrapper around `localStorage` that returns `null` on read failures and returns `false` on write failures (instead of throwing).

```js
App.Storage = {
  get(key)        → any | null,     // JSON.parse, returns null on error
  set(key, value) → boolean,        // JSON.stringify, returns false on QuotaExceededError
  remove(key)     → void
}
```

**LocalStorage keys:**

| Key                    | Value type | Description                                              |
|------------------------|------------|----------------------------------------------------------|
| `tld_tasks`            | Task[]     | Serialized task array                                    |
| `tld_links`            | Link[]     | Serialized link array                                    |
| `tld_theme`            | string     | `"light"` or `"dark"`                                   |
| `tld_username`         | string     | User's display name                                      |
| `tld_stats`            | StatsData  | Productivity statistics (streak, totals, last-streak-date) |
| `tld_timer_duration`   | number     | Custom Pomodoro duration in minutes                      |

**KEYS constants** (all keys are registered as named constants inside `App.Storage.KEYS`):

```js
const KEYS = {
  TASKS:          'tld_tasks',
  LINKS:          'tld_links',
  THEME:          'tld_theme',
  USERNAME:       'tld_username',
  STATS:          'tld_stats',
  TIMER_DURATION: 'tld_timer_duration',
};
```

---

### App.Stats

**Responsibilities:** Track and display productivity statistics (total completed, today's completed, daily streak). Reacts to task completion/uncompletion events emitted by `App.TodoList`. Persists via `App.Storage` using `KEYS.STATS`.

**Public interface:**

```js
App.Stats.init()            // called once on DOMContentLoaded
App.Stats.onTaskChanged()   // called by App.TodoList after any toggleComplete
```

**Internal state:**

```js
{
  total:    number,         // all-time completed count (≥ 0)
  today:    number,         // tasks completed on the current calendar day (≥ 0)
  streak:   number,         // consecutive-day streak (≥ 0)
  lastDate: string | null   // ISO date string "YYYY-MM-DD" of last streak increment
}
```

**Streak logic (pure function):**

```js
function computeStreak(state, todayStr) → { streak: number, lastDate: string }
```

Rules:
- `todayStr` = today's date as `"YYYY-MM-DD"` (derived from `new Date().toISOString().slice(0, 10)`)
- If `state.lastDate === todayStr` → no change (streak already counted today)
- If `state.lastDate` equals yesterday's ISO date → `streak + 1`, `lastDate = todayStr`
- Otherwise (`lastDate` is `null` or older than yesterday) → `streak = 1`, `lastDate = todayStr`
- Applied only on the **first** completion of the current calendar day

**On load (stale streak check — Req 13.10):** After reading `tld_stats`, if `lastDate` is neither today nor yesterday, reset `streak` to `0` before rendering.

**`onTaskChanged` logic:**

1. Recount `total` and `today` from the live task array obtained from `App.TodoList` (or passed in via the call).
2. If a task was just **completed** and today's completed count just incremented from 0 to 1, call `computeStreak` and update `streak` / `lastDate`.
3. If a task was just **uncompleted**, decrement `total` and `today` (each floored at 0); leave `streak` unchanged.
4. Persist via `App.Storage.set(App.Storage.KEYS.STATS, state)`.
5. Re-render the `#widget-stats` DOM.

---

### App.GreetingWidget

**Responsibilities:** Display clock (HH:MM), date (human-readable), contextual greeting, custom name input.

**Public interface:**

```js
App.GreetingWidget.init()   // called once on DOMContentLoaded
```

**Internal state:**

```js
{ userName: string | null }
```

**Clock tick:** `setInterval(tick, 1000)` — updates time display every second and triggers date update on minute change.

**Greeting logic (pure function):**

```js
function getGreeting(hour) → "Good Morning" | "Good Afternoon" | "Good Evening" | "Good Night"
```

| Hour range  | Greeting        |
|-------------|-----------------|
| 05 – 11     | Good Morning    |
| 12 – 17     | Good Afternoon  |
| 18 – 20     | Good Evening    |
| 21 – 04     | Good Night      |

---

### App.FocusTimer

**Responsibilities:** Countdown timer with start/stop/reset controls, configurable duration, completion notification.

**Public interface:**

```js
App.FocusTimer.init()   // called once on DOMContentLoaded
```

**Internal state (updated for custom duration — Req 14):**

```js
{
  durationMinutes: number,   // 5–60, default 25 (persisted via KEYS.TIMER_DURATION)
  remaining:       number,   // seconds — initialized to durationMinutes × 60
  running:         boolean,
  intervalId:      number | null
}
```

**Updated `reset()` behavior:** Restores `remaining` to `state.durationMinutes × 60` (not always `1500`), in accordance with Req 14.5.

**Duration change handler (new — Req 14.3, 14.4, 14.6, 14.7):**

Triggered on `blur` or `Enter` on the `#timer-duration` numeric input:

1. Read raw value from input.
2. Parse as integer; clamp to `[5, 60]`: `Math.max(5, Math.min(60, parsed))`.
3. Set `state.durationMinutes = clamped`.
4. Persist via `App.Storage.set(App.Storage.KEYS.TIMER_DURATION, clamped)`.
5. If timer is currently running: call `stop()` first (sets `running = false`).
6. Call `reset()` to apply the new duration to `remaining` and update the display.

**New storage key:** `App.Storage.KEYS.TIMER_DURATION` (`"tld_timer_duration"`).

**On load:** Read `tld_timer_duration`; if the stored value is a number in `[5, 60]` use it, otherwise default to `25`. Initialize `remaining = durationMinutes × 60`.

---

### App.TodoList

**Responsibilities:** Add/edit/delete/complete tasks, sort tasks, persist to localStorage, notify `App.Stats` on completion changes.

**Public interface:**

```js
App.TodoList.init()   // called once on DOMContentLoaded
```

**Internal state:**

```js
{
  tasks: Task[],
  sortOrder: "default" | "completed-last",
  editingId: string | null
}
```

**Task operations (all mutate in-memory state, then persist, then re-render):**

```js
addTask(label)           → void
editTask(id, newLabel)   → void
toggleComplete(id)       → void
deleteTask(id)           → void
setSortOrder(order)      → void
```

**Updated `addTask` flow (Req 15):**

1. Call `validateLabel(label)` — reject if invalid (empty/whitespace/too long).
2. **Duplicate check:** `state.tasks.some(t => t.label.trim().toLowerCase() === trimmed.toLowerCase())` — if `true`, set `#todo-error` to `"Task already exists."` and return without creating the task (Req 15.1, 15.2).
3. If no duplicate: create task object, push to `state.tasks`, persist, re-render.

**Stats integration (Req 13.2):** After `toggleComplete(id)` mutates the task and re-renders, call `App.Stats.onTaskChanged()` so all three productivity metrics are updated immediately.

**Validation (pure functions):**

```js
validateLabel(raw)       → { valid: boolean, trimmed: string, error?: string }
```

---

### App.QuickLinks

**Responsibilities:** Add/delete links, open URLs in new tab, persist to localStorage.

**Public interface:**

```js
App.QuickLinks.init()   // called once on DOMContentLoaded
```

**Internal state:**

```js
{ links: Link[] }
```

**Link operations:**

```js
addLink(label, url)      → void
deleteLink(id)           → void
```

**Validation (pure function):**

```js
validateLink(label, url) → { valid: boolean, error?: string }
```

---

### App.ThemeManager

**Responsibilities:** Toggle light/dark theme via a CSS class on `<html>`, persist choice.

**Public interface:**

```js
App.ThemeManager.init()
App.ThemeManager.setTheme(theme)   // "light" | "dark"
```

Theme is applied by toggling `data-theme="dark"` on the `<html>` element. CSS custom properties handle all visual switching.

---

## Data Models

### Task

```js
{
  id:        string,   // crypto.randomUUID() or Date.now().toString() fallback
  label:     string,   // 1 – 255 characters (trimmed)
  completed: boolean,  // false on creation
  createdAt: number    // Date.now() — preserves insertion order
}
```

### Link

```js
{
  id:    string,   // crypto.randomUUID() or Date.now().toString() fallback
  label: string,   // 1 – 50 characters (trimmed, truncated at 50)
  url:   string,   // must start with "http://" or "https://"
  valid: boolean   // computed at load time: url starts with http(s)://
}
```

### Theme

```js
type Theme = "light" | "dark"   // stored as string in localStorage
```

### UserName

```js
type UserName = string | null   // null when not set; 1 – 50 chars when set
```

### StatsData

```js
// StatsData — persisted under App.Storage.KEYS.STATS ("tld_stats")
{
  total:    number,         // all-time completed count (≥ 0)
  today:    number,         // tasks completed on the current calendar day (≥ 0)
  streak:   number,         // consecutive-day streak (≥ 0)
  lastDate: string | null   // ISO "YYYY-MM-DD" of last streak increment, or null
}
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Greeting matches hour

*For any* hour value (0–23), the greeting function SHALL return exactly one of the four defined greeting strings, and the returned string SHALL match the hour range defined in Requirements 1.3–1.6 — no hour is unhandled and no hour maps to two different greetings.

**Validates: Requirements 1.3, 1.4, 1.5, 1.6**

---

### Property 2: Task label validation rejects blank

*For any* string composed entirely of whitespace characters (including the empty string), `validateLabel` SHALL return `{ valid: false }`, and the task list SHALL remain unchanged after a rejected submission.

**Validates: Requirements 3.3, 4.3**

---

### Property 3: Task add round-trip preserves data

*For any* valid task label (1–255 non-whitespace-only characters), after `addTask(label)` is called, the resulting in-memory task array SHALL contain exactly one Task whose trimmed label equals the input label, and `App.Storage.get("tld_tasks")` SHALL deserialize to an array containing that same Task with its `completed` flag set to `false`.

**Validates: Requirements 3.2, 5.1**

---

### Property 4: Task edit preserves identity and updates label

*For any* existing Task and any valid new label, after `editTask(id, newLabel)` is called, the task array SHALL contain a Task with the same `id` and `createdAt` as before, with `label` equal to the trimmed new label.

**Validates: Requirements 4.2**

---

### Property 5: Task completion toggle is an involution

*For any* Task, calling `toggleComplete(id)` twice in succession SHALL leave the Task's `completed` state identical to what it was before either call.

**Validates: Requirements 4.5**

---

### Property 6: Completed-last sort places all incomplete tasks before all completed tasks

*For any* task array, after applying the "completed-last" sort, every Task with `completed === false` SHALL appear at a lower index than every Task with `completed === true`. Tasks within the same completion group SHALL preserve their relative insertion order.

**Validates: Requirements 11.2, 11.3**

---

### Property 7: localStorage round-trip preserves task collection

*For any* array of Task objects, serializing via `App.Storage.set("tld_tasks", tasks)` and then deserializing via `App.Storage.get("tld_tasks")` SHALL produce an array equal to the original (same length, same `id`, `label`, `completed`, `createdAt` for each element).

**Validates: Requirements 5.1, 5.2**

---

### Property 8: Link URL validation enforces http(s) prefix

*For any* URL string, `validateLink` SHALL return `{ valid: true }` if and only if the URL starts with `"http://"` or `"https://"`. All other strings SHALL produce `{ valid: false }`.

**Validates: Requirements 7.2, 7.4**

---

### Property 9: Link label truncation stays within bound

*For any* label string of arbitrary length, after `addLink(label, url)` is called with a valid URL, the stored Link's label SHALL have length ≤ 50.

**Validates: Requirements 7.6**

---

### Property 10: Theme persistence round-trip

*For any* theme value `"light"` or `"dark"`, after `ThemeManager.setTheme(theme)` is called, `App.Storage.get("tld_theme")` SHALL return that same theme value.

**Validates: Requirements 9.3, 9.4**

---

### Property 11: Username persistence round-trip

*For any* non-empty username string (1–50 characters, trimmed), after the user submits it, `App.Storage.get("tld_username")` SHALL return the trimmed value, and the greeting SHALL include that name.

**Validates: Requirements 10.2, 10.4**

---

### Property 12: Stats counts never go negative

*For any* sequence of `toggleComplete` calls on any task list (regardless of initial completion states, order of toggling, or how many times a task is uncompleted), `state.total` and `state.today` in `App.Stats` SHALL never drop below zero.

**Validates: Requirements 13.8**

---

### Property 13: Streak never double-increments on the same calendar day

*For any* `App.Stats` state and any number of task completions that all occur on the same calendar day (i.e., `computeStreak` is called multiple times with the same `todayStr`), the `streak` value SHALL be incremented at most once relative to its value at the start of that day.

**Validates: Requirements 13.5**

---

### Property 14: Duplicate task detection is case- and whitespace-insensitive

*For any* pair of strings `(a, b)` such that `a.trim().toLowerCase() === b.trim().toLowerCase()`, if a task with label `a` already exists in the task list then submitting label `b` SHALL always be rejected with an error message, and the task count SHALL remain unchanged.

**Validates: Requirements 15.1, 15.2**

---

### Property 15: Custom Pomodoro duration clamping

*For any* integer input `n` entered into the duration field, the applied duration stored in `state.durationMinutes` and persisted to `App.Storage.KEYS.TIMER_DURATION` SHALL equal `Math.max(5, Math.min(60, n))`. The displayed and effective timer duration SHALL never be outside the range `[5, 60]` minutes.

**Validates: Requirements 14.1, 14.4**

---

## Error Handling

### localStorage Failures

`App.Storage.set` catches `QuotaExceededError` and `SecurityError` and returns `false`. Callers check the return value and surface a non-blocking inline message near the relevant widget (e.g., "⚠ Could not save — storage may be full."). The in-memory state is not rolled back — the user's session remains functional.

`App.Storage.get` catches `SyntaxError` (malformed JSON) and returns `null`. Callers treat `null` as "no data" and use the appropriate default (empty array, `"light"` theme, no username).

### Input Validation Failures

Validation errors are shown as inline text messages adjacent to the relevant input (not modal alerts). Messages are cleared on the next successful submission. No `alert()` or `confirm()` calls are used.

### Invalid URLs at Load Time

When loading Quick Links from storage, each URL is checked for `http://` or `https://` prefix. Links that fail are rendered with a `disabled` attribute and a distinct visual style (reduced opacity, `not-allowed` cursor). They are retained in storage — not silently deleted.

### Timer Edge Cases

- Calling start when already running is a no-op (idempotent guard).
- Calling stop when not running is a no-op.
- Reset always clears the interval unconditionally before restoring state.

---

## Testing Strategy

This feature is a client-side vanilla JS application. It contains a mix of **pure logic functions** well-suited to property-based testing and **DOM/UI behaviors** better covered by example-based unit tests.

### Property-Based Testing

**Library:** [fast-check](https://fast-check.dev/) (loaded from CDN in the test harness HTML, or via `npx jest` with `fast-check` as a dev dependency if a minimal test setup is desired).

Each correctness property (Properties 1–11 above) maps to one property-based test. Tests are configured to run **100 iterations minimum** per property.

**Tag format for test comments:**
```
// Feature: todo-life-dashboard, Property N: <property text>
```

**Properties most valuable for PBT:**

| Property | What varies | Why PBT adds value |
|----------|-------------|-------------------|
| 1 (Greeting) | Hour 0–23 | Ensures exhaustive coverage of boundary hours (5, 12, 18, 21, 0) |
| 2 (Label rejection) | Whitespace strings of all lengths | Catches edge cases: `" "`, `"\t"`, `"\n\n"`, zero-length |
| 3 (Task round-trip) | Label content, length, Unicode | Catches serialization bugs with special chars, max-length labels |
| 5 (Toggle involution) | Any task state | Ensures toggle is truly reversible |
| 6 (Completed-last sort) | Arrays of mixed-completion tasks | Verifies stable ordering across all permutations |
| 7 (localStorage round-trip) | Task arrays with varied content | Validates JSON serialize/deserialize correctness |
| 8 (URL validation) | Arbitrary URL strings | Catches edge cases: `ftp://`, `javascript:`, `//`, empty |
| 9 (Label truncation) | Label strings of length 0–200 | Ensures no label ever exceeds 50 chars |
| 12 (Stats non-negative) | Task lists with many toggle sequences | Catches off-by-one and floor-at-zero bugs |
| 13 (Streak no double-increment) | Many completions on same day | Validates `computeStreak` idempotence within a day |
| 14 (Duplicate detection) | All case/whitespace string pairs | Catches normalization edge cases (leading spaces, UPPER, mixed) |
| 15 (Duration clamping) | All integers including negatives and large numbers | Validates clamp function at all boundaries |

### Unit / Example-Based Tests

For behaviors that do not benefit from input variation:

- **Timer state machine**: Verify transitions (idle→running, running→paused, paused→running, running→completed) with concrete examples.
- **Task deletion**: Verify deleted task is absent from the array.
- **Edit cancel**: Verify original label is preserved after Escape.
- **Empty task list placeholder**: Verify placeholder message renders when `tasks.length === 0`.
- **Empty links placeholder**: Verify placeholder message renders when `links.length === 0`.
- **Theme default**: Verify `"light"` is applied when no theme key is in storage.
- **Username default**: Verify greeting renders without a name when no username is in storage.
- **localStorage write failure**: Mock `localStorage.setItem` to throw, verify error message appears.
- **Stats load — valid data**: Verify `App.Stats` initializes `total`, `today`, `streak` from stored `StatsData`.
- **Stats load — missing/invalid data**: Verify all three metrics initialize to `0`.
- **Stats load — stale streak**: Verify `streak` is reset to `0` when `lastDate` is older than yesterday.
- **Duration load — saved value**: Verify timer initializes to the saved duration from `tld_timer_duration`.
- **Duration load — no saved value**: Verify timer defaults to `25` minutes.
- **Duration change while running**: Verify timer stops and resets to new duration.
- **Duration change while stopped**: Verify display updates without starting countdown.
- **Duplicate rejection — error message**: Verify `#todo-error` shows `"Task already exists."` on exact-match duplicate.
- **Stats write failure**: Mock `App.Storage.set` returning `false`; verify in-memory stats unchanged and no visible error.

### Testing Structure

Since there is no build tool, tests can be run in one of two ways:

1. **In-browser test runner**: A separate `test.html` file that loads `fast-check` from CDN and runs all tests, reporting results in a `<pre>` element.
2. **Node.js + Jest**: A minimal `package.json` with `jest` + `jsdom` + `fast-check` for CI-friendly runs. Pure-logic functions are extracted into a testable form by importing the script under `jsdom`.

The recommended approach for this project is **option 1** (in-browser), matching the no-build-tool constraint.

---

## DOM / HTML Additions

The following new HTML elements are required for Requirements 13–14. They follow the existing Glass Card / widget pattern and IIFE module conventions.

### Productivity Stats widget (`#widget-stats`)

Insert a new `<section>` **before** `#widget-todo` in the dashboard grid:

```html
<!-- ─── Productivity Stats Widget ─── -->
<section class="widget" id="widget-stats" aria-label="Productivity statistics">
  <p class="widget-title">Stats</p>
  <div class="stats-grid">
    <div class="stat-item">
      <span class="stat-label">All-time completed</span>
      <span class="stat-value" id="stats-total" aria-live="polite">0</span>
    </div>
    <div class="stat-item">
      <span class="stat-label">Completed today</span>
      <span class="stat-value" id="stats-today" aria-live="polite">0</span>
    </div>
    <div class="stat-item">
      <span class="stat-label">Daily streak</span>
      <span class="stat-value" id="stats-streak" aria-live="polite">0</span>
    </div>
  </div>
</section>
```

All three `<span>` value elements carry `aria-live="polite"` so screen readers announce metric changes after a task is toggled (Req 16.4).

### Focus Timer duration input (`#timer-duration`)

Add a numeric input row inside `#widget-timer`, rendered between the timer display and the controls row:

```html
<div class="timer-duration-row">
  <label for="timer-duration">Duration (min):</label>
  <input
    type="number"
    id="timer-duration"
    min="5"
    max="60"
    value="25"
    aria-label="Set focus timer duration in minutes"
  />
</div>
```

The input is wired by `App.FocusTimer.init()` — on `blur` and on `keydown Enter`, the duration change handler clamps, applies, and persists the value.

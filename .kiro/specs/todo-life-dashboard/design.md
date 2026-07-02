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
App.FocusTimer            ← countdown timer logic
App.TodoList              ← task list CRUD + sorting + persistence
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

| Key                   | Value type | Description                     |
|-----------------------|------------|---------------------------------|
| `tld_tasks`           | Task[]     | Serialized task array           |
| `tld_links`           | Link[]     | Serialized link array           |
| `tld_theme`           | string     | `"light"` or `"dark"`           |
| `tld_username`        | string     | User's display name             |

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

**Responsibilities:** 25-minute countdown, start/stop/reset controls, completion notification.

**Public interface:**

```js
App.FocusTimer.init()   // called once on DOMContentLoaded
```

**Internal state:**

```js
{
  remaining: number,    // seconds remaining (0 – 1500)
  running: boolean,
  intervalId: number | null
}
```

**Timer tick:** `setInterval(tick, 1000)` while running. Each tick decrements `remaining` by 1 and re-renders. At `remaining === 0`, the timer stops and shows a completion message.

---

### App.TodoList

**Responsibilities:** Add/edit/delete/complete tasks, sort tasks, persist to localStorage.

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

### Testing Structure

Since there is no build tool, tests can be run in one of two ways:

1. **In-browser test runner**: A separate `test.html` file that loads `fast-check` from CDN and runs all tests, reporting results in a `<pre>` element.
2. **Node.js + Jest**: A minimal `package.json` with `jest` + `jsdom` + `fast-check` for CI-friendly runs. Pure-logic functions are extracted into a testable form by importing the script under `jsdom`.

The recommended approach for this project is **option 1** (in-browser), matching the no-build-tool constraint.

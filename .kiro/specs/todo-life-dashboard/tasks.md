# Implementation Plan: To-Do Life Dashboard

## Overview

Implement a self-contained single-page web application (`index.html`) with embedded CSS and JavaScript. The app delivers five widgets — Greeting, Focus Timer, To-Do List, Quick Links, and Theme Toggle — all running in vanilla ES6+ JavaScript with no build tooling. All data is persisted via `localStorage`. Tests are written in a companion `test.html` file using fast-check loaded from CDN.

---

## Tasks

- [x] 1. Scaffold the single-file application structure
  - Create `index.html` with `<head>`, `<body>`, and `<script>` sections
  - Add `<style>` block with CSS custom properties for light/dark themes and base widget layout
  - Add `window.App = {}` namespace at the top of the `<script>` block
  - Add stub IIFE wrappers for each module: `App.Storage`, `App.GreetingWidget`, `App.FocusTimer`, `App.TodoList`, `App.QuickLinks`, `App.ThemeManager`
  - Add a `DOMContentLoaded` listener that calls each module's `init()` function in order
  - Add semantic HTML markup placeholders for all five widget sections
  - _Requirements: (all — foundational structure)_

- [x] 2. Implement App.Storage
  - [x] 2.1 Implement the `App.Storage` module
    - Implement `get(key)`: `JSON.parse` from `localStorage`, return `null` on `SyntaxError` or missing key
    - Implement `set(key, value)`: `JSON.stringify` to `localStorage`, catch `QuotaExceededError` / `SecurityError`, return `false` on failure and `true` on success
    - Implement `remove(key)`: call `localStorage.removeItem(key)`
    - Define all storage key constants: `tld_tasks`, `tld_links`, `tld_theme`, `tld_username`
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 8.1, 8.2, 8.3, 8.4, 9.3, 9.4, 10.4_

  - [ ]* 2.2 Write property test for localStorage round-trip (Property 7)
    - **Property 7: localStorage round-trip preserves task collection**
    - For any array of Task objects, `set` then `get` on `tld_tasks` returns an array equal in length, `id`, `label`, `completed`, and `createdAt`
    - **Validates: Requirements 5.1, 5.2**

- [x] 3. Implement App.ThemeManager
  - [x] 3.1 Implement the `App.ThemeManager` module
    - Implement `setTheme(theme)`: toggle `data-theme` attribute on `<html>`, call `App.Storage.set("tld_theme", theme)`
    - Implement `init()`: read theme from storage (default `"light"` if absent or storage unavailable), call `setTheme` before any content renders
    - Wire the theme toggle button's `click` event to switch between `"light"` and `"dark"` and call `setTheme`
    - Apply theme within 200 ms via synchronous DOM attribute write (no animation delay needed for attribute toggle)
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6_

  - [ ]* 3.2 Write property test for theme persistence round-trip (Property 10)
    - **Property 10: Theme persistence round-trip**
    - For any value `"light"` or `"dark"`, after `setTheme(theme)`, `App.Storage.get("tld_theme")` returns the same value
    - **Validates: Requirements 9.3, 9.4**

- [x] 4. Implement App.GreetingWidget
  - [x] 4.1 Implement the greeting pure function and clock
    - Implement `getGreeting(hour)` pure function: hour 5–11 → "Good Morning", 12–17 → "Good Afternoon", 18–20 → "Good Evening", 21–4 → "Good Night"
    - Implement `tick()`: read `new Date()` from the system clock, update the time display (HH:MM), update the date display on minute change (handles midnight rollover)
    - Start `setInterval(tick, 1000)` in `init()`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8_

  - [ ]* 4.2 Write property test for greeting-matches-hour (Property 1)
    - **Property 1: Greeting matches hour**
    - For any integer hour 0–23, `getGreeting(hour)` returns exactly one of the four strings and matches the correct range boundary
    - **Validates: Requirements 1.3, 1.4, 1.5, 1.6**

  - [x] 4.3 Implement the custom username input
    - Add a text input (max 50 chars) and confirm button to the Greeting Widget HTML
    - On confirm (button click or Enter key): trim input, validate non-empty, call `App.Storage.set("tld_username", trimmed)`, update greeting display
    - On empty / whitespace-only submission: call `App.Storage.remove("tld_username")`, display greeting without name
    - In `init()`: read `tld_username` from storage and render greeting with name if present, without name if absent or storage unavailable
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7_

  - [ ]* 4.4 Write property test for username persistence round-trip (Property 11)
    - **Property 11: Username persistence round-trip**
    - For any non-empty trimmed string 1–50 chars, after user submits it, `App.Storage.get("tld_username")` returns the trimmed value and the greeting includes that name
    - **Validates: Requirements 10.2, 10.4**

- [x] 5. Implement App.FocusTimer
  - [x] 5.1 Implement the Focus Timer state machine and rendering
    - Define internal state: `{ remaining: 1500, running: false, intervalId: null }`
    - Implement `start()`: guard if already running (no-op), set `running = true`, start `setInterval(tick, 1000)`; each tick decrements `remaining` by 1, re-renders, and at 0 calls `complete()`
    - Implement `stop()`: guard if not running (no-op), clear interval, set `running = false`, re-render
    - Implement `reset()`: unconditionally clear interval, restore state to `{ remaining: 1500, running: false, intervalId: null }`, re-render
    - Implement `complete()`: clear interval, set `running = false`, render completion message
    - Implement `render()`: format `remaining` as MM:SS and update the display element
    - Wire start/stop/reset button `click` events in `init()`
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7, 2.8, 2.9_

- [x] 6. Implement App.TodoList — core logic and validation
  - [x] 6.1 Implement task validation and data helpers
    - Implement `validateLabel(raw)`: trim input, return `{ valid: false, error: "..." }` for empty/whitespace-only or length > 255; return `{ valid: true, trimmed }` otherwise
    - Implement `createTask(label)`: return a Task object `{ id: crypto.randomUUID(), label, completed: false, createdAt: Date.now() }` (with `Date.now().toString()` fallback for id)
    - _Requirements: 3.2, 3.3, 4.2, 4.3_

  - [ ]* 6.2 Write property test for label validation rejects blank (Property 2)
    - **Property 2: Task label validation rejects blank**
    - For any string composed entirely of whitespace (including empty string), `validateLabel` returns `{ valid: false }`
    - **Validates: Requirements 3.3, 4.3**

  - [x] 6.3 Implement `addTask`, `editTask`, `toggleComplete`, `deleteTask`
    - `addTask(label)`: call `validateLabel`, show inline error on failure; on success push new Task to `state.tasks`, call `App.Storage.set("tld_tasks", state.tasks)` and check return value (show save-error message if `false`), then re-render
    - `editTask(id, newLabel)`: call `validateLabel`; on failure silently retain original label and exit edit mode; on success update matching task's label, persist, re-render
    - `toggleComplete(id)`: flip `completed` flag on matching task, persist, re-render (re-apply current sort order if not default)
    - `deleteTask(id)`: filter task out of array, persist, re-render
    - _Requirements: 3.2, 3.3, 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 5.1, 5.4_

  - [ ]* 6.4 Write property test for task add round-trip (Property 3)
    - **Property 3: Task add round-trip preserves data**
    - For any valid label, after `addTask(label)`, the in-memory array contains exactly one task with the trimmed label and `completed === false`, and `App.Storage.get("tld_tasks")` deserializes to an array containing that task
    - **Validates: Requirements 3.2, 5.1**

  - [ ]* 6.5 Write property test for task edit preserves identity (Property 4)
    - **Property 4: Task edit preserves identity and updates label**
    - For any existing task and valid new label, after `editTask(id, newLabel)`, the task with the same `id` and `createdAt` has its label updated to the trimmed new label
    - **Validates: Requirements 4.2**

  - [ ]* 6.6 Write property test for toggle involution (Property 5)
    - **Property 5: Task completion toggle is an involution**
    - For any task, calling `toggleComplete(id)` twice returns `completed` to its original value
    - **Validates: Requirements 4.5**

- [ ] 7. Implement App.TodoList — rendering and edit mode
  - [ ] 7.1 Implement `render()` and inline edit mode
    - `render()`: if `state.tasks` is empty show placeholder message; otherwise build task items using `getSortedTasks()` — each item has a checkbox (reflects `completed`), label text with strikethrough when `completed === true`, edit button, and delete button
    - When edit button is clicked, replace label with a pre-filled `<input>`, confirm on Enter / blur, cancel on Escape; set `state.editingId`
    - Use event delegation on the task list container for all click and keyboard events
    - _Requirements: 3.4, 3.5, 4.1, 4.4, 4.6, 4.7_

  - [ ] 7.2 Implement task sorting (completed-last)
    - Implement `getSortedTasks()`: if `state.sortOrder === "default"` return tasks sorted by `createdAt` ascending; if `"completed-last"` return incomplete tasks (sorted by `createdAt`) followed by completed tasks (sorted by `createdAt`)
    - Add sort control (select or button group) to the HTML with options "Default" and "Completed Last", defaulting to "Default"
    - On sort control change, update `state.sortOrder` and re-render within 300 ms
    - _Requirements: 11.1, 11.2, 11.3, 11.4_

  - [ ]* 7.3 Write property test for completed-last sort (Property 6)
    - **Property 6: Completed-last sort places all incomplete before all completed**
    - For any mixed-completion task array, after applying "completed-last", every `completed === false` task appears at a lower index than every `completed === true` task, with relative insertion order preserved within each group
    - **Validates: Requirements 11.2, 11.3**

- [ ] 8. Implement App.TodoList — persistence and load
  - [ ] 8.1 Implement load-from-storage in `TodoList.init()`
    - In `init()`: call `App.Storage.get("tld_tasks")`, treat `null` or non-array as empty list (no error surfaced to user), set `state.tasks`, then call `render()`
    - _Requirements: 5.2, 5.3_

- [ ] 9. Checkpoint — Core widgets functional
  - Ensure all tests pass, ask the user if questions arise.

- [x] 10. Implement App.QuickLinks — core logic and validation
  - [x] 10.1 Implement link validation and `addLink`, `deleteLink`
    - Implement `validateLink(label, url)`: return `{ valid: false, error: "Label required" }` if label is empty after trim; return `{ valid: false, error: "URL must start with http:// or https://" }` if URL does not start with `"http://"` or `"https://"`; otherwise return `{ valid: true }`
    - `addLink(label, url)`: call `validateLink`, show inline error on failure; on success truncate label to 50 chars, create Link object `{ id: crypto.randomUUID(), label: truncated, url, valid: true }`, push to `state.links`, persist, re-render
    - `deleteLink(id)`: filter link out of array, persist, re-render
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 8.1, 8.4_

  - [ ]* 10.2 Write property test for URL validation (Property 8)
    - **Property 8: Link URL validation enforces http(s) prefix**
    - For any URL string, `validateLink` returns `{ valid: true }` iff the URL starts with `"http://"` or `"https://"`; all other strings produce `{ valid: false }`
    - **Validates: Requirements 7.2, 7.4**

  - [ ]* 10.3 Write property test for label truncation (Property 9)
    - **Property 9: Link label truncation stays within bound**
    - For any label string of arbitrary length with a valid URL, after `addLink`, the stored link's label has length ≤ 50
    - **Validates: Requirements 7.6**

- [ ] 11. Implement App.QuickLinks — rendering and persistence
  - [ ] 11.1 Implement `render()` and load-from-storage
    - `render()`: if `state.links` is empty show placeholder message; otherwise render each link as a `<button>` with label text truncated with ellipsis if > 50 chars, `target="_blank"` navigation on click
    - For links with `valid === false`, render button with `disabled` attribute and reduced-opacity / `not-allowed` cursor style; do not open new tab on click
    - In `init()`: `App.Storage.get("tld_links")`, treat `null` or non-array as empty (no error), compute `valid` flag for each loaded link, set `state.links`, call `render()`
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 8.2, 8.3_

- [ ] 12. Create the in-browser test harness
  - [ ] 12.1 Create `test.html` with fast-check loaded from CDN
    - Create `test.html` that loads fast-check from CDN (`https://cdn.jsdelivr.net/npm/fast-check/+esm` or equivalent unpkg URL)
    - Import or inline the pure-logic functions under test (`getGreeting`, `validateLabel`, `validateLink`, `getSortedTasks`, `App.Storage` with a mock `localStorage`)
    - Add a `<pre id="results">` element where each test appends pass/fail lines
    - _Requirements: (testing infrastructure for all properties)_

  - [ ]* 12.2 Write all 11 property-based tests in test.html
    - Implement PBT for Properties 1–11 using fast-check arbitraries (minimum 100 runs each)
    - Tag each test with `// Feature: todo-life-dashboard, Property N: <description>`
    - **Property 1**: `fc.integer({min:0,max:23})` → greeting in correct range
    - **Property 2**: whitespace-only strings → `validateLabel` returns `{ valid: false }`
    - **Property 3**: valid labels → `addTask` round-trip in memory and storage
    - **Property 4**: existing task + valid new label → `editTask` preserves `id` and `createdAt`
    - **Property 5**: any task → double `toggleComplete` restores original state
    - **Property 6**: mixed arrays → completed-last sort invariant holds
    - **Property 7**: task arrays → `Storage.set` then `Storage.get` round-trip equality
    - **Property 8**: arbitrary URL strings → `validateLink` result matches `http(s)://` prefix check
    - **Property 9**: arbitrary label lengths → stored label ≤ 50 chars
    - **Property 10**: `"light"` | `"dark"` → `Storage.get("tld_theme")` returns same value
    - **Property 11**: valid usernames → storage and greeting inclusion
    - **Validates: Requirements 1.3–1.6, 3.2, 3.3, 4.2, 4.3, 4.5, 5.1, 5.2, 7.2, 7.4, 7.6, 9.3, 9.4, 10.2, 10.4, 11.2, 11.3**

- [ ] 13. Final checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties defined in Properties 1–11 of the design
- Unit / example-based tests (timer state machine, deletion, edit-cancel, placeholders, defaults) should be added to `test.html` alongside the property tests
- The single-file delivery constraint (`index.html`) means no imports — all module code must be written inside the `<script>` block in dependency order (Storage → ThemeManager → GreetingWidget → FocusTimer → TodoList → QuickLinks)

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1"] },
    { "id": 1, "tasks": ["3.1", "4.1", "5.1", "6.1"] },
    { "id": 2, "tasks": ["3.2", "4.2", "4.3", "6.2", "6.3", "10.1"] },
    { "id": 3, "tasks": ["4.4", "6.4", "6.5", "6.6", "7.1", "7.2", "10.2", "10.3", "11.1"] },
    { "id": 4, "tasks": ["7.3", "8.1", "2.2"] },
    { "id": 5, "tasks": ["12.1"] },
    { "id": 6, "tasks": ["12.2"] }
  ]
}
```

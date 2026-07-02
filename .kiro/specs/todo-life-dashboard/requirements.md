# Requirements Document

## Introduction

The To-Do Life Dashboard is a client-side web application that serves as a personal daily homepage. It displays the current time and date with a contextual greeting, a Pomodoro-style focus timer, a persistent to-do list, and a collection of quick-access links to favorite websites. All data is stored in the browser's Local Storage — no backend or server setup is required. Bonus features include light/dark mode toggle, a customizable greeting name, and task sorting.

## Glossary

- **Dashboard**: The single-page web application rendered in the browser.
- **Greeting_Widget**: The UI component that displays the current time, date, and a personalized greeting message.
- **Focus_Timer**: The UI component that implements a 25-minute countdown timer with start, stop, and reset controls.
- **Todo_List**: The UI component that manages a user's tasks, including add, edit, complete, and delete operations.
- **Quick_Links**: The UI component that displays user-defined shortcut buttons that open URLs in the browser.
- **Local_Storage**: The browser's Web Storage API used for all client-side data persistence.
- **Task**: A single to-do item with a text label and a completion state.
- **Link**: A user-defined record consisting of a display label and a URL.
- **Theme**: The visual color scheme applied to the Dashboard, either light or dark.
- **User_Name**: The custom name entered by the user, displayed in the greeting.
- **Glass_Card**: A UI container panel styled with the frosted-glass effect (semi-transparent fill, backdrop blur, soft shadow, rounded corners) used to visually group each feature section.
- **Accent_Color**: A single designated color used exclusively for interactive highlights such as primary buttons, active states, and focus indicators.

---

## Requirements

### Requirement 1: Greeting Widget

**User Story:** As a user, I want to see the current time, date, and a greeting based on the time of day, so that I am oriented when I open the dashboard.

#### Acceptance Criteria

1. THE Greeting_Widget SHALL display the current time in HH:MM format, updated every second.
2. THE Greeting_Widget SHALL display the current date in a human-readable format (e.g., "Monday, June 29, 2026").
3. WHEN the current hour is between 05:00 and 11:59, THE Greeting_Widget SHALL display the greeting "Good Morning".
4. WHEN the current hour is between 12:00 and 17:59, THE Greeting_Widget SHALL display the greeting "Good Afternoon".
5. WHEN the current hour is between 18:00 and 20:59, THE Greeting_Widget SHALL display the greeting "Good Evening".
6. WHEN the current hour is between 21:00 and 04:59, THE Greeting_Widget SHALL display the greeting "Good Night".
7. THE Greeting_Widget SHALL use the browser's local system clock as the time source for all time and date values.
8. WHEN the displayed minute changes, THE Greeting_Widget SHALL update the date display to reflect the current date (handling midnight rollovers).

---

### Requirement 2: Focus Timer

**User Story:** As a user, I want a 25-minute countdown timer with start, stop, and reset controls, so that I can focus on tasks using the Pomodoro technique.

#### Acceptance Criteria

1. THE Focus_Timer SHALL initialize with a countdown value of 25 minutes and 00 seconds (25:00).
2. WHEN the user activates the start control, THE Focus_Timer SHALL begin counting down one second at a time.
3. WHILE the Focus_Timer is counting down, THE Focus_Timer SHALL update the displayed time every second.
4. WHEN the user activates the stop control, THE Focus_Timer SHALL pause the countdown at the current value.
5. WHEN the user activates the reset control, THE Focus_Timer SHALL stop the countdown and restore the display to 25:00.
6. WHEN the countdown reaches 00:00, THE Focus_Timer SHALL stop automatically and display a visible completion message on the page.
7. THE Focus_Timer SHALL display the remaining time in MM:SS format at all times (counting down, paused, reset, and completed states).
8. WHEN the user activates the start control while the timer is already counting down, THE Focus_Timer SHALL continue the countdown uninterrupted without resetting.
9. WHEN the user activates the stop control while the timer is not counting down, THE Focus_Timer SHALL make no state change.

---

### Requirement 3: To-Do List — Add and Display Tasks

**User Story:** As a user, I want to add tasks to a list and see them displayed, so that I can track what I need to do.

#### Acceptance Criteria

1. THE Todo_List SHALL provide a text input field and an "Add" control for creating new tasks.
2. WHEN the user submits a non-empty task label, THE Todo_List SHALL trim leading and trailing whitespace from the label, then append a new Task to the list with the trimmed label and a completion state of false. The label SHALL NOT exceed 255 characters.
3. IF the user submits a label that is empty or becomes empty after trimming whitespace, THEN THE Todo_List SHALL reject the submission and display a validation message.
4. THE Todo_List SHALL display each Task with its label text and a distinct visual control (e.g., checkbox) that reflects the current completion state — appearing checked when complete and unchecked when incomplete.
5. WHEN the task list is empty, THE Todo_List SHALL display a placeholder message indicating that no tasks have been added.

---

### Requirement 4: To-Do List — Edit, Complete, and Delete Tasks

**User Story:** As a user, I want to edit, mark as done, and delete tasks, so that I can keep my list accurate and up to date.

#### Acceptance Criteria

1. WHEN the user activates the edit control on a Task, THE Todo_List SHALL replace the Task's label display with an editable text input pre-filled with the current label value.
2. WHEN the user confirms an edit with a non-empty label (after trimming whitespace), THE Todo_List SHALL update the Task's label to the trimmed value (maximum 255 characters) and exit edit mode.
3. IF the user confirms an edit with a label that is empty or becomes empty after trimming, THEN THE Todo_List SHALL reject the edit, retain the original label, and exit edit mode.
4. WHEN the user cancels an edit (e.g., presses Escape), THE Todo_List SHALL discard the changes, retain the original label, and exit edit mode.
5. WHEN the user activates the complete control on a Task, THE Todo_List SHALL toggle the Task's completion state between true and false.
6. WHILE a Task's completion state is true, THE Todo_List SHALL apply a strikethrough style to that Task's label.
7. WHEN the user activates the delete control on a Task, THE Todo_List SHALL remove that Task from the list and it SHALL NOT be recoverable through any UI action.

---

### Requirement 5: To-Do List — Persistence

**User Story:** As a user, I want my tasks to be saved automatically, so that my list is preserved when I close or refresh the browser.

#### Acceptance Criteria

1. WHEN any Task is added, edited, completed, or deleted, THE Todo_List SHALL write the complete updated task collection to Local_Storage before the next user interaction can be processed.
2. WHEN the Dashboard loads, THE Todo_List SHALL read the task collection from Local_Storage and render all previously saved Tasks with their label and completion state.
3. IF no task data exists in Local_Storage or the stored data cannot be parsed as a valid task collection, THEN THE Todo_List SHALL render an empty list without error.
4. IF writing to Local_Storage fails, THEN THE Todo_List SHALL display an error message indicating that the task could not be saved, and the in-memory task list SHALL remain unchanged.

---

### Requirement 6: Quick Links — Display and Navigation

**User Story:** As a user, I want to see shortcut buttons for my favorite websites, so that I can navigate to them quickly from the dashboard.

#### Acceptance Criteria

1. THE Quick_Links SHALL display each saved Link as a labeled button showing the Link's display name, truncated with an ellipsis if the name exceeds 50 characters.
2. WHEN the user activates a Link button, THE Quick_Links SHALL open the associated URL in a new browser tab.
3. WHEN the Quick_Links collection is empty, THE Quick_Links SHALL display a placeholder message indicating that no links have been added.
4. IF a saved Link's URL is malformed or invalid at load time, THEN THE Quick_Links SHALL render the button in a visually disabled state and SHALL NOT open a new tab when the button is activated.

---

### Requirement 7: Quick Links — Add and Delete Links

**User Story:** As a user, I want to add and remove quick-link buttons, so that I can customize the sites available on my dashboard.

#### Acceptance Criteria

1. THE Quick_Links SHALL provide a text input field for a display label (maximum 50 characters), a text input field for a URL, and an "Add" control.
2. WHEN the user submits a Link with a non-empty label and a URL that begins with "http://" or "https://", THE Quick_Links SHALL add the new Link to the collection.
3. IF the user submits a Link with an empty label, THEN THE Quick_Links SHALL reject the submission and display a validation message indicating the label is required.
4. IF the user submits a Link with a URL that does not begin with "http://" or "https://", THEN THE Quick_Links SHALL reject the submission and display a validation message indicating the URL is invalid.
5. WHEN the user activates the delete control on a Link, THE Quick_Links SHALL remove that Link from the collection and it SHALL NOT be recoverable through any UI action.
6. WHEN the user submits a Link with a label longer than 50 characters, THE Quick_Links SHALL truncate the label to 50 characters before adding it to the collection.

---

### Requirement 8: Quick Links — Persistence

**User Story:** As a user, I want my quick links to be saved automatically, so that my shortcuts are preserved when I close or refresh the browser.

#### Acceptance Criteria

1. WHEN any Link is added or deleted, THE Quick_Links SHALL write the updated link collection to Local_Storage.
2. WHEN the Dashboard loads, THE Quick_Links SHALL read the link collection from Local_Storage and render all previously saved Links.
3. IF no link data exists in Local_Storage or the stored data cannot be parsed as a valid link collection, THEN THE Quick_Links SHALL render an empty links section without error.
4. IF writing to Local_Storage fails, THEN THE Quick_Links SHALL display a visible, non-blocking error message indicating that the link could not be saved.

---

### Requirement 9: Light / Dark Mode (Bonus)

**User Story:** As a user, I want to toggle between light and dark visual themes, so that I can use the dashboard comfortably in different lighting conditions.

#### Acceptance Criteria

1. THE Dashboard SHALL provide a toggle control that switches between exactly two Theme options: light and dark.
2. WHEN the user activates the theme toggle, THE Dashboard SHALL apply the selected Theme to all UI components within 200 milliseconds without a page reload.
3. WHEN the Dashboard loads, THE Dashboard SHALL read the saved Theme from Local_Storage and apply it before rendering content.
4. WHEN the user changes the Theme, THE Dashboard SHALL write the selected Theme value to Local_Storage.
5. IF no Theme value exists in Local_Storage, THEN THE Dashboard SHALL apply the light Theme as the default.
6. IF Local_Storage is unavailable (e.g., browser private mode or storage quota exceeded), THEN THE Dashboard SHALL apply the light Theme as the default and SHALL NOT surface a storage error to the user.

---

### Requirement 10: Custom Name in Greeting (Bonus)

**User Story:** As a user, I want to set my name in the greeting, so that the dashboard feels personalized.

#### Acceptance Criteria

1. THE Greeting_Widget SHALL provide a text input control (maximum 50 characters) and a confirm control (button or Enter key) for the user to enter and save a User_Name.
2. WHEN the user submits a non-empty User_Name (between 1 and 50 characters, after trimming whitespace), THE Greeting_Widget SHALL include the User_Name in the greeting (e.g., "Good Morning, Alex!").
3. WHEN the Dashboard loads, THE Greeting_Widget SHALL read the User_Name from Local_Storage and display it in the greeting if present.
4. WHEN the user submits a User_Name via the confirm control, THE Greeting_Widget SHALL write the trimmed User_Name to Local_Storage.
5. IF no User_Name is saved in Local_Storage, THEN THE Greeting_Widget SHALL display the greeting without a name (e.g., "Good Morning!").
6. WHEN the user submits an empty User_Name (or whitespace-only), THE Greeting_Widget SHALL remove the User_Name from Local_Storage and display the greeting without a name.
7. IF Local_Storage is unavailable, THEN THE Greeting_Widget SHALL display the greeting without a name and SHALL NOT surface a storage error to the user.

---

### Requirement 11: Sort Tasks (Bonus)

**User Story:** As a user, I want to sort my task list, so that I can prioritize incomplete tasks or view completed ones separately.

#### Acceptance Criteria

1. THE Todo_List SHALL provide a sort control that defaults to "Default" (insertion order) on initial load and offers at least two sort options: "Default" and "Completed Last" (incomplete tasks first, completed tasks last).
2. WHEN the user selects a sort option, THE Todo_List SHALL re-render the task list in the selected order within 300 milliseconds.
3. THE Todo_List SHALL preserve each Task's label and completion state regardless of the active sort order.
4. WHEN a Task's completion state changes while a non-default sort order is active, THE Todo_List SHALL re-apply the current sort order to the updated list.

---

### Requirement 12: UI/UX — Liquid Glass Design System

**User Story:** As a user, I want the dashboard to have a calm, modern glassmorphism visual style, so that the interface feels focused, lightweight, and visually cohesive across light and dark modes.

#### Acceptance Criteria

1. THE Dashboard SHALL render all feature sections (Greeting_Widget, Focus_Timer, Todo_List, Quick_Links) inside individual card panels styled with a frosted-glass effect: a semi-transparent background with opacity between 10% and 30%, a backdrop blur filter between 12px and 24px, a soft drop shadow, and no visible border exceeding 1px in width.
2. THE Dashboard SHALL apply rounded corners between 12px and 20px to all glass card panels.
3. THE Dashboard SHALL use a single-column centered layout with consistent vertical spacing between 16px and 32px between card panels to provide visual breathing room between sections.
4. THE Dashboard SHALL apply a neutral base background color — white or near-white in light Theme, dark or near-black in dark Theme — as the backdrop behind all glass card panels.
5. THE Dashboard SHALL use a single accent color for interactive highlights such as primary buttons, active states, and focus indicators, and SHALL NOT apply multiple high-saturation colors across the interface.
6. WHILE the light Theme is active, THE Dashboard SHALL render glass card panels with a white semi-transparent fill at 15%–25% opacity and a soft drop shadow with a blur radius of 8px–20px, such that the frosted-glass effect remains clearly visible.
7. WHILE the dark Theme is active, THE Dashboard SHALL render glass card panels with a dark semi-transparent fill at 10%–20% opacity and a soft outer glow with a blur radius of 8px–20px, such that the frosted-glass effect remains clearly visible.
8. THE Dashboard SHALL display text using a clean sans-serif typeface — either the system-default sans-serif stack or the Inter typeface — and SHALL NOT use decorative or serif fonts for any UI element.
9. THE Dashboard SHALL apply a clear typographic hierarchy of at least three distinct size levels: the time display in the Greeting_Widget SHALL be rendered at a minimum of 48px, section labels and primary content SHALL be rendered at a minimum of 20px, and supporting text SHALL be rendered at a minimum of 14px.
10. THE Dashboard SHALL render all interactive controls (buttons, inputs, toggles) in a subtle, low-visual-weight style that avoids heavy borders, high-contrast fills, or blocky shapes inconsistent with the glassmorphism aesthetic.
11. WHEN the Dashboard first renders, THE Dashboard SHALL apply a gentle fade-in transition to each glass card panel with a duration between 200ms and 400ms using ease-out easing, so that the cards appear progressively rather than abruptly.
12. WHEN the user hovers over a glass card panel or an interactive control within a panel, THE Dashboard SHALL apply a smooth hover effect — a vertical lift between 2px and 4px via CSS transform and/or a backdrop blur increase of no more than 8px — completing within 150 milliseconds.
13. THE Dashboard SHALL NOT apply any animation or transition with a repeating cycle shorter than 333 milliseconds (equivalent to ≥ 3Hz), and SHALL NOT use flashing, strobing, or pulsing effects that distract from content readability.

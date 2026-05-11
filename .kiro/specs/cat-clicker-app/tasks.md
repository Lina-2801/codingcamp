# Implementation Plan: Cat Clicker App

## Overview

Build a single-page cat clicker game as one HTML file (`index.html`) with an optional co-located `app.js`. The implementation follows a vanilla JS MVC pattern (AppState → DOMView → AppController), uses Tailwind CSS via CDN, and includes an inline SVG cat graphic with CSS `@keyframes` animations. Tests are written with Jest + jsdom (unit) and fast-check (property-based), plus Playwright for end-to-end smoke tests.

---

## Tasks

- [~] 1. Set up project structure and test environment
  - Create `index.html` shell with Tailwind CDN `<script>` tag, semantic HTML skeleton (`<header>`, `<main>`, `<footer>`), and all required DOM element IDs: `cat`, `score-value`, `milestone-message`, `reset-btn`
  - Create `app.js` with empty module stubs for `AppState`, `DOMView`, and `AppController`
  - Create `tests/` directory with empty `model.test.js`, `view.test.js`, `controller.test.js`, and `e2e.spec.js` files
  - Add `package.json` with `jest`, `jest-environment-jsdom`, `fast-check`, and `@playwright/test` as dev dependencies; configure Jest to use jsdom environment
  - _Requirements: 7.1_

- [ ] 2. Implement the Model (`AppState`)
  - [-] 2.1 Implement `AppState` object with `clickCount`, `triggeredMilestones` (Set), `MILESTONES` constant, and `ANIMATION_DURATIONS` constant
    - Implement `incrementClick()`: adds 1 to `clickCount`
    - Implement `resetState()`: sets `clickCount` to 0, clears `triggeredMilestones`
    - Implement `checkMilestone(count)`: returns milestone value if `count` is in `MILESTONES` and not yet in `triggeredMilestones`, else `null`; adds to `triggeredMilestones` when returning a value
    - _Requirements: 2.1, 4.1, 5.2, 5.6_

  - [~] 2.2 Write property test for Click Count Accuracy (Property 1)
    - **Property 1: Click Count Accuracy**
    - Generate random starting count (0–9999) and random N (1–200); call `incrementClick` N times; assert `clickCount === start + N`
    - Use `fc.integer({min:0, max:9999})` and `fc.integer({min:1, max:200})`
    - Tag: `// Feature: cat-clicker-app, Property 1: Click Count Accuracy`
    - **Validates: Requirements 2.1, 2.4, 7.3**
    - _File: tests/model.test.js_

  - [~] 2.3 Write property test for Reset Restores Initial State (Property 4)
    - **Property 4: Reset Restores Initial State**
    - Generate random `clickCount` (0–9999) and random subset of triggered milestones; call `resetState()`; assert `clickCount === 0` and `triggeredMilestones` is empty
    - Use `fc.integer({min:0, max:9999})` and `fc.subarray([10,50,100,500,1000])`
    - Tag: `// Feature: cat-clicker-app, Property 4: Reset Restores Initial State`
    - **Validates: Requirements 5.2, 5.5, 5.6**
    - _File: tests/model.test.js_

  - [~] 2.4 Write unit tests for `AppState`
    - Test `checkMilestone` returns correct value at each of the 5 milestones
    - Test `checkMilestone` returns `null` for a milestone already in `triggeredMilestones`
    - Test `checkMilestone` returns `null` for non-milestone counts
    - Test `resetState` clears a partially-filled `triggeredMilestones` set
    - _Requirements: 2.1, 4.1, 5.2, 5.6_
    - _File: tests/model.test.js_

- [ ] 3. Implement the View (`DOMView`)
  - [~] 3.1 Implement `DOMView` object with DOM element references and `updateScore(count)`
    - Query and store `scoreEl`, `catEl`, `messageEl`, `resetBtn` by their IDs
    - Implement `updateScore(count)`: coerce with `Math.max(0, Math.floor(n))`, set `scoreEl.textContent`
    - _Requirements: 3.1, 3.2, 3.3_

  - [~] 3.2 Implement `playCatAnimation(type)`, `showMilestoneMessage(count)`, `hideMilestoneMessage()`, and `resetView()`
    - `playCatAnimation`: removes all animation classes, forces reflow (`offsetWidth`), adds the requested class (`.cat--click`, `.cat--celebrate`, `.cat--idle`); uses `setTimeout` to restore `.cat--idle` after the animation duration; checks `CSS.supports` for graceful degradation
    - `showMilestoneMessage`: sets message text to `"You reached {N} clicks!"`, makes `messageEl` visible, clears any pending dismiss timer, starts a 2000 ms `setTimeout` calling `hideMilestoneMessage`
    - `hideMilestoneMessage`: hides `messageEl`, clears pending dismiss timer
    - `resetView`: calls `updateScore(0)`, `hideMilestoneMessage()`, calls `playCatAnimation('idle')`
    - _Requirements: 2.2, 4.2, 4.3, 4.4, 4.5, 5.3, 5.4, 5.5_

  - [~] 3.3 Write property test for Score Display Accuracy (Property 2)
    - **Property 2: Score Display Accuracy**
    - Generate random integer 0–999999; call `updateScore(n)`; assert `scoreEl.textContent === String(n)`
    - Use `fc.integer({min:0, max:999999})`
    - Tag: `// Feature: cat-clicker-app, Property 2: Score Display Accuracy`
    - **Validates: Requirements 3.2**
    - _File: tests/view.test.js_

  - [~] 3.4 Write unit tests for `DOMView`
    - Test score display label ("Clicks:" or "Score:") is present in the DOM
    - Test `hideMilestoneMessage` hides the element
    - Test `showMilestoneMessage` sets correct text and auto-dismisses after 2000 ms (use `jest.useFakeTimers`)
    - Test milestone message replacement: show message for 10, then show for 50 before 2000 ms; verify text shows "50" and timer resets
    - Test `playCatAnimation('click')` adds `.cat--click` and removes it after `ANIMATION_DURATIONS.click` ms
    - Test `playCatAnimation('celebrate')` adds `.cat--celebrate`
    - Test graceful degradation: mock `CSS.supports` to return `false`; call `playCatAnimation`; verify no animation classes are toggled
    - _Requirements: 2.2, 3.2, 4.2, 4.4, 4.5, 7.3_
    - _File: tests/view.test.js_

- [~] 4. Checkpoint — Ensure all model and view tests pass
  - Run `npx jest tests/model.test.js tests/view.test.js` and confirm all tests pass. Ask the user if any questions arise.

- [ ] 5. Implement the Controller (`AppController`) and wire everything together
  - [~] 5.1 Implement `AppController` with `handleCatClick()`, `handleReset()`, and `init()`
    - `handleCatClick`: calls `AppState.incrementClick()`, calls `DOMView.updateScore(AppState.clickCount)`, calls `DOMView.playCatAnimation('click')`, calls `AppState.checkMilestone(AppState.clickCount)` and if non-null calls `DOMView.playCatAnimation('celebrate')` and `DOMView.showMilestoneMessage(milestone)`
    - `handleReset`: calls `AppState.resetState()`, calls `DOMView.resetView()`
    - `init`: attaches `click` listener on `catEl` → `handleCatClick`, attaches `click` listener on `resetBtn` → `handleReset`, calls `DOMView.updateScore(0)` and `DOMView.playCatAnimation('idle')` for initial render
    - Wire `init()` call inside a `DOMContentLoaded` listener in `app.js` (or inline `<script>` in `index.html`)
    - _Requirements: 2.1, 2.3, 2.4, 4.2, 4.3, 5.2, 5.3, 5.4, 5.5, 5.6_

  - [~] 5.2 Write property test for Milestone Message Content (Property 3)
    - **Property 3: Milestone Message Content**
    - For each milestone M in {10, 50, 100, 500, 1000}: set `AppState.clickCount` to M−1, call `AppController.handleCatClick()`, assert `messageEl.textContent` includes `String(M)`
    - Use `fc.constantFrom(10, 50, 100, 500, 1000)`
    - Tag: `// Feature: cat-clicker-app, Property 3: Milestone Message Content`
    - **Validates: Requirements 4.2**
    - _File: tests/controller.test.js_

  - [~] 5.3 Write unit tests for `AppController`
    - Test `handleCatClick` increments score and updates DOM
    - Test `handleCatClick` while click animation in progress still increments count (animation restart)
    - Test `handleReset` sets score display to 0 and dismisses any active message
    - Test `handleReset` cancels active animation and restores `.cat--idle` within 300 ms
    - _Requirements: 2.1, 2.3, 2.4, 5.2, 5.3, 5.4, 5.5_
    - _File: tests/controller.test.js_

- [ ] 6. Build the HTML structure and inline SVG cat graphic
  - [~] 6.1 Write the full `index.html` layout with Tailwind utility classes
    - Score display: sticky/fixed header with label and `<span id="score-value">0</span>`; font size ≥ 16 px; visible at all viewport widths 320–2560 px
    - Cat section: horizontally centered `<div id="cat">` containing the inline SVG; SVG `viewBox` set so rendered width is 150–400 px via CSS constraints; clickable hit area ≥ 44×44 px
    - Milestone message: `<div id="milestone-message">` hidden by default (Tailwind `hidden` class); fixed/absolute position overlay
    - Reset button: `<button id="reset-btn">` with minimum 44×44 px touch target; legible at all viewport widths
    - Responsive single-column stacking below 480 px viewport width
    - _Requirements: 1.1, 1.2, 3.1, 5.1, 6.1, 6.2, 6.3, 6.4_

  - [~] 6.2 Create the inline SVG cat graphic with cartoon styling
    - SVG with `viewBox="0 0 200 200"` (or similar), bold outlines, flat/pastel colors, rounded shapes
    - Include distinct body parts (head, ears, eyes, tail) as named groups or elements to support per-element animation targeting
    - Minimum bounding box 150×150 px enforced via CSS on the wrapper `<div id="cat">`
    - _Requirements: 1.1, 1.2, 6.2_

- [ ] 7. Implement CSS animations
  - [~] 7.1 Add `@keyframes` rules and animation classes to `index.html` (or a `<style>` block)
    - `.cat--idle`: gentle vertical float + tail sway, 2 s loop (`animation-duration: 2000ms`, `animation-iteration-count: infinite`)
    - `.cat--click`: scale up to 1.15 then back to 1.0 (squish), `animation-duration: 250ms`, `animation-iteration-count: 1`
    - `.cat--celebrate`: spin + scale pulse, visually distinct from both idle and click, `animation-duration: 800ms`, `animation-iteration-count: 1`
    - Add `@media (prefers-reduced-motion: reduce)` block that sets `animation: none` for all three classes
    - _Requirements: 1.3, 1.4, 2.2, 4.3, 7.3_

- [~] 8. Checkpoint — Full integration check
  - Run `npx jest` to confirm all unit and property tests pass. Ask the user if any questions arise.

- [ ] 9. Write Playwright end-to-end tests
  - [~] 9.1 Write e2e smoke tests in `tests/e2e.spec.js`
    - Open `index.html` via `file://` protocol in Playwright; verify page loads without JS console errors
    - Verify cat element is visible and has rendered width ≥ 150 px
    - Click the cat 10 times; verify score display shows "10" and milestone message contains "10"
    - Click reset; verify score display shows "0" and milestone message is hidden
    - At viewport widths 320, 480, 768, 1440, 2560 px — verify no horizontal scrollbar (`document.body.scrollWidth <= window.innerWidth`)
    - Verify cat hit area and reset button meet 44×44 px minimum (`getBoundingClientRect`)
    - _Requirements: 1.1, 2.1, 3.2, 4.2, 5.2, 6.1, 6.2, 6.3, 7.2_
    - _File: tests/e2e.spec.js_

- [~] 10. Final checkpoint — All tests pass
  - Run `npx jest` and `npx playwright test` and confirm all tests pass. Ask the user if any questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- Checkpoints (tasks 4, 8, 10) ensure incremental validation at natural breaks
- Property tests validate universal correctness guarantees; unit tests cover specific scenarios and edge cases
- The `app.js` file is optional — all JS can live in an inline `<script>` tag at the bottom of `index.html`
- No build step or dev server is needed; open `index.html` directly in a browser via `file://`

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1"] },
    { "id": 1, "tasks": ["2.2", "2.3", "2.4", "3.1"] },
    { "id": 2, "tasks": ["3.2"] },
    { "id": 3, "tasks": ["3.3", "3.4", "5.1"] },
    { "id": 4, "tasks": ["5.2", "5.3", "6.1"] },
    { "id": 5, "tasks": ["6.2", "7.1"] },
    { "id": 6, "tasks": ["9.1"] }
  ]
}
```

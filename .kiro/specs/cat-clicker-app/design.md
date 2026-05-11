# Design Document: Cat Clicker App

## Overview

The Cat Clicker App is a single-page browser game built with HTML5, Tailwind CSS (CDN), and vanilla JavaScript. The entire application lives in one HTML file (with an optional co-located `.js` file) and runs via the `file://` protocol — no server, no build step, no dependencies beyond the Tailwind CDN script.

The core gameplay loop is simple: the user clicks a cartoonized cat graphic to accumulate points. Visual feedback is provided at every click (animation), at milestone thresholds (celebration + congratulatory message), and at all times via a persistent score display. A reset button returns the game to its initial state.

### Key Design Goals

- **Zero infrastructure**: pure static file, CDN-only external dependency.
- **Progressive enhancement**: the app is fully functional even when CSS animations are disabled.
- **Responsive by default**: fluid layout from 320 px to 2560 px with no horizontal overflow.
- **Accessible**: touch targets ≥ 44 × 44 px, legible font sizes, semantic HTML.

---

## Architecture

The app follows a minimal **Model → View → Controller** pattern implemented entirely in vanilla JavaScript within a single HTML file.

```
┌─────────────────────────────────────────────────────┐
│                     index.html                      │
│                                                     │
│  ┌──────────────┐   events   ┌──────────────────┐  │
│  │  Controller  │◄──────────►│      View        │  │
│  │  (JS logic)  │            │  (DOM + CSS)     │  │
│  └──────┬───────┘            └──────────────────┘  │
│         │ read/write                                │
│  ┌──────▼───────┐                                  │
│  │    Model     │                                  │
│  │  (JS object) │                                  │
│  └──────────────┘                                  │
└─────────────────────────────────────────────────────┘
```

**Data flow:**
1. User clicks/taps the cat → DOM `click` event fires.
2. Controller increments the model's `clickCount`.
3. Controller checks milestone conditions and updates `triggeredMilestones`.
4. Controller calls View helpers to update the score display, trigger animations, and show/hide the congratulatory message.

No framework, no virtual DOM, no reactive state library — all DOM mutations are explicit and synchronous (except animation class toggling which relies on CSS transitions).

---

## Components and Interfaces

### 1. Model (`AppState`)

A plain JavaScript object (or class) that holds all mutable state. It is the single source of truth.

```js
const AppState = {
  clickCount: 0,                  // non-negative integer
  triggeredMilestones: new Set(), // milestone values already fired this session
};
```

**Interface:**

| Function | Signature | Description |
|---|---|---|
| `incrementClick` | `() → void` | Adds 1 to `clickCount` |
| `resetState` | `() → void` | Sets `clickCount` to 0, clears `triggeredMilestones` |
| `checkMilestone` | `(count: number) → number \| null` | Returns the milestone value if `count` is a milestone not yet triggered, else `null` |

---

### 2. View (`DOMView`)

Responsible for all DOM reads and writes. Never mutates `AppState` directly.

```js
const DOMView = {
  scoreEl:       /* <span id="score-value"> */,
  catEl:         /* <div id="cat"> */,
  messageEl:     /* <div id="milestone-message"> */,
  resetBtn:      /* <button id="reset-btn"> */,
};
```

**Interface:**

| Function | Signature | Description |
|---|---|---|
| `updateScore` | `(count: number) → void` | Sets `scoreEl.textContent` to `count` |
| `playCatAnimation` | `(type: 'click' \| 'celebrate' \| 'idle') → void` | Adds/removes CSS animation classes on `catEl` |
| `showMilestoneMessage` | `(count: number) → void` | Renders message text, makes `messageEl` visible, starts 2000 ms auto-dismiss timer |
| `hideMilestoneMessage` | `() → void` | Hides `messageEl`, clears any pending dismiss timer |
| `resetView` | `() → void` | Calls `updateScore(0)`, `hideMilestoneMessage()`, transitions cat to idle |

---

### 3. Controller (`AppController`)

Wires the Model and View together. Attaches event listeners on `DOMContentLoaded`.

**Interface:**

| Function | Signature | Description |
|---|---|---|
| `handleCatClick` | `() → void` | Increments model, updates view score, triggers click animation, checks milestone |
| `handleReset` | `() → void` | Resets model, resets view |
| `init` | `() → void` | Queries DOM elements, attaches event listeners, renders initial state |

---

### 4. Cat Graphic (SVG)

The cat is an inline SVG embedded directly in the HTML. Using inline SVG (rather than an `<img>`) allows CSS classes and animations to target SVG child elements (eyes, body, tail) for richer idle/celebration effects.

Minimum bounding box: 150 × 150 px. The SVG uses `viewBox` so it scales fluidly via CSS `width`/`height` constraints.

---

### 5. Animation System

All animations are implemented as CSS `@keyframes` rules toggled by JavaScript class manipulation. This ensures graceful degradation: if animations are disabled (e.g., `prefers-reduced-motion`, or the browser strips CSS), the cat simply renders in its static end-state.

| Animation Class | Trigger | Duration | Keyframe Effect |
|---|---|---|---|
| `.cat--idle` | Default / after any animation ends | 2 s loop | Gentle vertical float + tail sway |
| `.cat--click` | On cat click | 250 ms | Scale up to 1.15 then back to 1.0 (squish) |
| `.cat--celebrate` | On milestone | 800 ms | Spin + scale pulse, visually distinct from click |

JavaScript removes the active animation class after its duration using `setTimeout`, then re-adds `.cat--idle`.

---

### 6. Milestone Message Overlay

A fixed-position `<div>` (hidden by default via `hidden` Tailwind class or `display:none`). When a milestone fires:
1. Its text is set to `"You reached {N} clicks!"`.
2. It becomes visible (CSS transition for fade-in).
3. A `setTimeout` of 2000 ms schedules `hideMilestoneMessage()`.
4. If a new milestone fires before the timer expires, the existing timer is cleared (`clearTimeout`) and a new one is started.

---

## Data Models

### AppState (runtime, in-memory only)

```ts
interface AppState {
  clickCount: number;           // integer ≥ 0
  triggeredMilestones: Set<number>; // subset of {10, 50, 100, 500, 1000}
}
```

### Milestone Definition (constant)

```js
const MILESTONES = [10, 50, 100, 500, 1000]; // ascending order
```

### Animation Duration Constants

```js
const ANIMATION_DURATIONS = {
  click:     250,  // ms — must be ≤ 300 ms per Req 2.2
  celebrate: 800,  // ms — must be ≤ 1000 ms per Req 4.3
  idle:      2000, // ms per cycle — within 1000–3000 ms per Req 1.4
  message:   2000, // ms auto-dismiss — per Req 4.4
};
```

### DOM Element IDs (contract between HTML and JS)

| ID | Element | Purpose |
|---|---|---|
| `cat` | `<div>` / SVG wrapper | Clickable cat graphic |
| `score-value` | `<span>` | Displays current click count |
| `milestone-message` | `<div>` | Congratulatory message overlay |
| `reset-btn` | `<button>` | Resets game state |

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*


### Property 1: Click Count Accuracy

*For any* starting click count and any sequence of N valid click events, the resulting click count SHALL equal the starting count plus N, regardless of whether a click animation is already in progress when subsequent clicks arrive.

**Validates: Requirements 2.1, 2.4, 7.3**

---

### Property 2: Score Display Accuracy

*For any* non-negative integer value passed to `updateScore`, the Score_Display element's rendered text SHALL equal that integer with no decimal places and no extraneous characters.

**Validates: Requirements 3.2**

---

### Property 3: Milestone Message Content

*For any* milestone value M in {10, 50, 100, 500, 1000}, when the click count reaches M and M has not yet been triggered in the current session, the displayed Congratulatory_Message text SHALL contain the string representation of M.

**Validates: Requirements 4.2**

---

### Property 4: Reset Restores Initial State

*For any* application state (arbitrary click count, arbitrary subset of triggered milestones, with or without an active congratulatory message), calling `resetState` SHALL produce a state where: (a) `clickCount` is 0, (b) `triggeredMilestones` is empty so all milestones are re-triggerable, and (c) any active Congratulatory_Message is dismissed.

**Validates: Requirements 5.2, 5.5, 5.6**

---

## Error Handling

### Animation Interruption

When a click arrives while a `click` or `celebrate` animation is in progress, the controller must:
1. Clear the existing `setTimeout` that would restore the idle class.
2. Remove the current animation class from the cat element.
3. Force a reflow (read `offsetWidth`) to restart the CSS animation.
4. Re-add the animation class and set a fresh `setTimeout`.

This prevents animation class accumulation and ensures the counter always increments.

### Milestone Timer Collision

When a new milestone fires while a `milestone-message` dismiss timer is pending:
1. Call `clearTimeout` on the stored timer ID.
2. Update the message text.
3. Start a new 2000 ms timer.

The timer ID is stored in a module-level variable (`let dismissTimer = null`) so it is always accessible for cancellation.

### Rapid Click Handling

The click handler is synchronous and does not debounce. Every click event increments the counter exactly once. There is no throttling — rapid taps on mobile are all counted.

### CSS Animation Unavailability

If `window.CSS` is undefined or `CSS.supports('animation-name', 'x')` returns `false`, the controller skips all animation class toggling. The cat renders in its static default state. Click counting and score display continue to function normally.

### Invalid State Guards

- `clickCount` is never allowed to go below 0. The reset function sets it to exactly 0.
- `updateScore` coerces its argument with `Math.max(0, Math.floor(n))` before rendering to guard against floating-point or negative values from unexpected callers.

---

## Testing Strategy

### Overview

This feature uses a **dual testing approach**: example-based unit tests for specific behaviors and property-based tests for universal correctness guarantees. The pure JavaScript logic layer (Model + Controller functions) is fully testable without a browser; View tests require a DOM environment (jsdom or a real browser).

### Property-Based Testing

**Library**: [fast-check](https://github.com/dubzzz/fast-check) (JavaScript, runs in Node.js with jsdom).

Each property test runs a **minimum of 100 iterations** with randomly generated inputs.

| Property | Test Description | Arbitraries |
|---|---|---|
| Property 1: Click Count Accuracy | Generate random starting count + random N (1–200), call `incrementClick` N times, assert final count = start + N | `fc.integer({min:0, max:9999})`, `fc.integer({min:1, max:200})` |
| Property 2: Score Display Accuracy | Generate random integer 0–999999, call `updateScore(n)`, assert `scoreEl.textContent === String(n)` | `fc.integer({min:0, max:999999})` |
| Property 3: Milestone Message Content | For each milestone M, set count to M, call `handleCatClick`, assert message text includes `String(M)` | `fc.constantFrom(10, 50, 100, 500, 1000)` |
| Property 4: Reset Restores Initial State | Generate random count + random triggered milestone subset, call `resetState`, assert count=0, milestones empty, message hidden | `fc.integer({min:0, max:9999})`, `fc.subarray([10,50,100,500,1000])` |

**Tag format for each test:**
```js
// Feature: cat-clicker-app, Property 1: Click Count Accuracy
// Feature: cat-clicker-app, Property 2: Score Display Accuracy
// Feature: cat-clicker-app, Property 3: Milestone Message Content
// Feature: cat-clicker-app, Property 4: Reset Restores Initial State
```

### Unit Tests (Example-Based)

Focus on specific scenarios, edge cases, and integration points:

- **Score display label**: verify the label text ("Clicks:" or "Score:") is present in the DOM.
- **Milestone auto-dismiss**: use fake timers (`jest.useFakeTimers`) to advance 2000 ms and verify message is hidden.
- **Milestone replacement**: trigger milestone 10, then milestone 50 before 2000 ms, verify message shows "50" and timer resets.
- **Animation class toggling**: verify `.cat--click` is added on click and removed after `ANIMATION_DURATIONS.click` ms.
- **Celebration animation**: verify `.cat--celebrate` is added when a milestone is reached.
- **Reset cancels animation**: verify animation classes are removed and `.cat--idle` is restored within 300 ms of reset.
- **Graceful degradation**: mock `CSS.supports` to return `false`, click the cat, verify count increments and no animation classes are toggled.
- **No horizontal overflow**: snapshot test verifying no element has `scrollWidth > clientWidth` at 320 px viewport width (jsdom).

### Integration / Smoke Tests

- **File protocol load**: open `index.html` via `file://` in a headless browser (Playwright), verify the page renders without JS errors.
- **Responsive layout**: at viewport widths 320 px, 480 px, 768 px, 1440 px, 2560 px — verify no horizontal scrollbar, cat width within 150–400 px range.
- **Touch target sizes**: verify cat hit area, reset button, and score display meet 44 × 44 px minimum.
- **CDN-only**: verify no `node_modules` directory, no `package.json` scripts referencing a dev server.

### Test File Structure

```
cat-clicker-app/
├── index.html
├── app.js              (optional co-located JS)
└── tests/
    ├── model.test.js   (unit + property tests for AppState)
    ├── view.test.js    (unit tests for DOMView with jsdom)
    ├── controller.test.js (unit + property tests for AppController)
    └── e2e.spec.js     (Playwright integration tests)
```

# Reflex: 3D Reaction Timer

A browser-based visual reaction time test with a 3D animated stage, built with vanilla JavaScript and [Three.js](https://threejs.org/). Start a round, wait for the shape to turn green, and react as fast as you can. Reflex tracks your session stats, flags false starts, and is designed to keep timing accurate even when the 3D graphics are under load.

The whole app is a single self-contained HTML file with no build step.

## Features

- **Accurate timing.** The green stimulus is timestamped in the same animation frame that draws it, and your response uses the browser's high-resolution event timestamp rather than the time the handler happened to run.
- **Unpredictable delays.** The wait before green is a random 1 to 5 seconds, drawn from `crypto.getRandomValues()` where available so it cannot be anticipated.
- **Fair scoring.** Pressing before green, or reacting in under 100 ms (faster than a genuine visual reaction allows), counts as a false start. No reaction within 3 seconds is a miss and isn't recorded.
- **Session statistics.** Best, average, last, slowest, valid attempts, false starts, and consistency (sample standard deviation).
- **3D visuals with graceful fallback.** A faceted icosahedron, wireframe shell, particle field, and shockwave effects respond to each state. If WebGL is unavailable or rendering fails, the app switches to a 2D mode without affecting timing.
- **Synthesized sound effects.** Generated with the Web Audio API (no audio files). There is deliberately no sound on the green signal, so this remains a pure visual test.
- **Adaptive performance.** A live FPS readout, and automatic resolution reduction if frames are consistently slow.
- **Accessible and responsive.** Keyboard play, screen reader announcements, `prefers-reduced-motion` support, device-aware instructions for touch and desktop, and safe-area handling on phones.

## Getting started

No installation is required.

1. Save the file as `index.html`.
2. Optionally add a `favicon.png` (128×128) in the same folder.
3. Open `index.html` in a modern browser.

To serve it locally instead of opening the file directly:

```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

An internet connection is needed on first load to fetch Three.js (r128, from cdnjs) and the Inter and Outfit fonts from Google Fonts. If Three.js fails to load, the app still runs in 2D mode, and the fonts fall back to system fonts.

## How to play

1. **Start a round.** Click or tap the stage, or press <kbd>Space</kbd>. The shape turns amber.
2. **Wait for green.** It changes after 1 to 5 seconds, at a random moment.
3. **React instantly.** Click, tap, or press <kbd>Space</kbd> as soon as it turns green. Early presses don't count.

### Controls

| Input | Action |
| --- | --- |
| <kbd>Space</kbd> | Start a round or react (works even when a button has focus) |
| <kbd>Enter</kbd> | Start or react when the stage has focus |
| <kbd>Esc</kbd> | Stop the current round |
| <kbd>M</kbd> | Toggle sound effects |
| Click / tap the stage | Start a round or react |
| **Start / Stop / Next attempt** button | Primary action for the current state |
| **Reset session** button | Clear all stats (Space never triggers this) |

## Game states

The app is driven by a small state machine. A single `data-state` attribute on `<body>` controls every state colour in the UI through CSS custom properties.

| State | Colour | Meaning |
| --- | --- | --- |
| `idle` | Slate | Waiting for the player to start |
| `waiting` | Amber | Round in progress, stimulus not yet shown |
| `ready` | Green | Stimulus shown, timing is running |
| `result` | Blue | Valid reaction recorded |
| `false-start` | Red | Early press, anticipation, or timeout |

### Rating scale

| Reaction time | Rating |
| --- | --- |
| under 180 ms | Exceptional reflexes |
| 180 to 219 ms | Excellent reaction |
| 220 to 259 ms | Great reaction |
| 260 to 309 ms | Good reaction |
| 310 to 399 ms | Average reaction |
| 400 ms and up | Room to improve |

## Configuration

Timing rules live in the frozen `CONFIG` object at the top of the script:

```js
const CONFIG = Object.freeze({
  MIN_DELAY_MS: 1000,   // shortest wait before green
  MAX_DELAY_MS: 5000,   // longest wait before green
  ANTICIPATION_MS: 100, // responses faster than this count as guesses
  TIMEOUT_MS: 3000,     // no response within this window is a miss
  COOLDOWN_MS: 350      // stops a double tap from starting a new round
});
```

Visual behaviour for each state (colour, glow, scale, spin speed, shell opacity, pulse) is defined in the `PROFILES` object, and the rating thresholds in the `rate()` function. Colours for the page itself are CSS custom properties in `:root` and the `body[data-state=...]` rules.

## How the timing works

Timing accuracy is the core concern, so the code is structured around a few rules:

- **The render loop owns the stimulus.** When the random delay expires, the timer only sets a `stimulusPending` flag. On the next `requestAnimationFrame` tick, the DOM and 3D scene switch to green, the frame is rendered, and only then is `stimulusAt` recorded with `performance.now()`. This keeps the timestamp tied to the frame that actually draws the change, rather than to a timer that might fire mid-frame.
- **The stimulus is never eased in.** CSS transitions and the 3D colour interpolation are bypassed for the `ready` state so green appears in a single frame.
- **Input uses event timestamps.** `eventTime()` prefers `event.timeStamp`, which shares the `performance.now()` time origin and excludes any delay before the handler ran. It falls back to `performance.now()` if the timestamp looks implausible.
- **Rendering can't break timing.** Any exception in the 3D update or render is caught, and the app drops to 2D mode for the rest of the session.
- **Hidden tabs cancel the round.** Browsers throttle animation frames in background tabs, so switching away during a round stops it.
- **Sounds play after timestamps.** Audio is only triggered once a time has been captured.

### Known limitations

Browser reaction tests measure the whole chain, not just your nervous system. Results include display latency, input device latency (Bluetooth keyboards and some touchscreens add noticeable delay), and the gap between when a frame is submitted and when it appears on screen. Scores are best compared against your own previous results on the same device rather than against published lab figures.

## Project structure

Everything is in one file:

| Section | Purpose |
| --- | --- |
| `<style>` | Layout, state colour tokens, responsive breakpoints, reduced-motion rules |
| Markup | Header, stage (canvas, fallback orb, overlay), controls, steps, session panel |
| `CONFIG`, `State` | Timing constants and state names |
| Utilities | Random delays, event timestamps, ratings, DOM helpers |
| `stats` | Pure session data: times, false starts, and derived values |
| `sound` | Web Audio synthesizer with lazy unlock on the first user gesture |
| `Visuals` class | Three.js scene, per-state profiles, shockwave, parallax, shake, resource disposal |
| State machine | `setState`, `startRound`, `recordResult`, `falseStart`, `handleAction` |
| Render loop | `frame()`: stimulus presentation, rendering, FPS and adaptive quality |
| Input | Pointer, mouse/touch fallback, click fallback, keyboard handling |
| Boot and teardown | Initialization, resize observation, cleanup on `pagehide` |

## Browser support

Works in current versions of Chrome, Edge, Firefox, and Safari on desktop and mobile. The app uses feature detection throughout and degrades gracefully:

- No WebGL, or Three.js fails to load: 2D mode with a CSS orb.
- No Pointer Events: mouse and touch listeners instead.
- No Web Audio: the sound button is disabled.
- No `ResizeObserver`: falls back to the window `resize` event.
- No `crypto.getRandomValues`: falls back to `Math.random()`.

WebGL context loss is handled, and all Three.js geometries, materials, and the renderer are disposed on teardown.

## Accessibility

- The stage is a focusable element with `role="button"` and a state-dependent label.
- Results, false starts, and state changes are announced through a polite live region.
- All actions are available from the keyboard, with visible focus outlines.
- `prefers-reduced-motion` disables pulsing, bobbing, parallax, camera shake, and UI animations, and is re-checked live if the setting changes.
- Instructions adapt to the device, for example "Tap the box" on touch screens and "Press Space or click" on desktop.

## Privacy

Session data is held in memory only. Nothing is stored in the browser or sent anywhere, and all stats clear when the page is closed.
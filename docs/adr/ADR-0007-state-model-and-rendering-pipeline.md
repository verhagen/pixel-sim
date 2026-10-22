# ADR-0007: State model and rendering pipeline

- **Status:** Accepted
- **Date:** 2026-10-22

## Context and Problem Statement

A real Pixoo is a state machine. Commands change its state (active channel, uploaded animation, text layers, brightness, rotation), and the firmware keeps turning that state into pixels: it advances animation frames and scrolls text. The simulator has to do the same, and has to show the result in one or more browser tabs and as PNG snapshots.

Where does the state live, who turns it into pixels, and how do we keep animation timing correct and testable?

## Decision Drivers

- One authoritative state, whether zero, one or several browser tabs are open.
- What the browser shows must match `/sim/frame.png` and the tests exactly.
- Animation and scrolling must be correct in time, and testable without real-time waiting.
- Bandwidth to the browser should be modest, even at animation frame rates.

## Considered Options

1. **Server-held state plus a pure, shared renderer** that runs both in the browser and on the server
2. The server renders every frame and streams pixels to the browser
3. The browser holds the state and renders it, and the server only forwards commands

## Decision Outcome

Chosen option: **1.**

### State

The server process keeps a single `DeviceState` object:

- `settings`: brightness, screen on/off, rotation, mirror, 24h flag, temperature unit, white balance, time zone, location, clock ID, custom page and so on. These are the values that `Channel/GetAllConf` and similar commands return.
- `channel`: Faces, Cloud, Visualizer, Custom, or *HTTP animation*. The last one is entered when an animation finishes uploading.
- `animation`: the active animation. It holds its frames, already scaled to 64×64 RGB, the per-frame duration and a `startedAt` timestamp.
- `pendingAnimations`: animations still being uploaded, keyed by `PicID`, plus the ID counter.
- `textLayers`: from `Draw/SendHttpText`, keyed by `TextId`, each with a `createdAt` timestamp.
- `items`: from `Draw/SendHttpItemList`, keyed by `TextId`.
- `indicators`: Tier 3 stub state such as timer, buzzer or scoreboard ([ADR-0004](ADR-0004-supported-command-set.md)).

Command handlers are the only code that changes state. Each change bumps a `revision` number and emits a change event ([ADR-0009](ADR-0009-web-ui-and-live-updates.md)).

### Renderer

A **pure function** in a shared module:

```ts
render(state: RenderableState, nowMs: number, env: RenderEnv): Uint8ClampedArray // 64*64*3 RGB
```

The pipeline runs in fixed order:

1. **Base layer:** the current frame of the active animation, chosen from `(nowMs - startedAt)` and the frame durations. For built-in channels (Faces/Cloud/Visualizer/Custom), a labelled placeholder (for example a simple clock for Faces).
2. **Overlays:** text layers and item-list items, drawn with bitmap fonts ([ADR-0008](ADR-0008-bitmap-fonts-for-text.md)). Scroll offsets come from `(nowMs - createdAt)` and `speed`. Dynamic items (time, date, temperature) are formatted from `env.now` and the simulated weather in `env`.
3. **Geometry:** rotation (0/90/180/270) and mirroring.
4. **Colour:** white balance (per-channel scale).
5. **Output:** brightness scaling, or all black when the screen is off.

Because the renderer is a pure function of state and time:

- The browser calls it on every `requestAnimationFrame` with its local clock, which gives smooth animation without streaming pixels.
- The server calls it on demand for `/sim/frame.png` and in tests.
- Tests pass explicit timestamps, for example "frame at t = 1250 ms", with no sleeps.

### Time

The server stamps state with its own clock. The browser estimates the offset between its clock and the server's from server timestamps in the event stream, so that animations line up across tabs. Dynamic items use the simulated device time, which `Device/SetUTC` and `Sys/TimeZone` can shift.

### Consequences

- Good: the browser view, PNG snapshots and tests come from the same code, with deterministic output.
- Good: only state changes are sent to the browser, not 60 frames per second of pixels.
- Good: renderer rules can be unit-tested in isolation.
- Bad: animation frames, up to `PicNum × 12 KB` of RGB, have to be sent to the browser once per upload. That is acceptable on localhost and on a LAN.
- Bad: the renderer has to stay environment-neutral (no Node or DOM APIs), which lint rules will enforce.

## More Information

### Open questions to verify on hardware

- The exact meaning and units of text `speed`, and whether short text that fits inside `TextWidth` still scrolls.
- The draw order between text layers and item-list items.
- Whether brightness is linear or gamma-corrected on the panel. The simulator starts with linear scaling.

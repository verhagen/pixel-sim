# ADR-0009: Web UI and live updates

- **Status:** Accepted
- **Date:** 2026-10-22

## Context and Problem Statement

People should *see* what their content looks like on the device ([ADR-0002](ADR-0002-simulator-vision-and-scope.md)). The simulator therefore needs a web page that shows the simulated display live, updated as soon as a client sends a command. Automated tests also need machine-readable access to the state and the rendered pixels.

How is the UI delivered and kept up to date, and what does it show?

## Decision Drivers

- No separate install. Opening a URL is enough.
- Updates should appear within a few milliseconds of a command, without polling.
- Easy to implement and debug with the standard library ([ADR-0006](ADR-0006-technology-stack.md)).
- The display should look like the real thing: individual round LEDs, not a blurry upscaled image.
- Automation should be able to read the frames and state without a browser.

## Considered Options

1. **Server-Sent Events (SSE)** carrying state changes, with the shared renderer in the browser
2. WebSockets
3. Browser polling of `/sim/state`
4. A desktop window (Electron or a native GUI)

## Decision Outcome

Chosen option: **1, SSE**, served by the simulator process itself.

Updates only travel from server to browser, so SSE is enough. It runs over plain HTTP with `node:http`, needs no extra library, and the browser reconnects automatically. WebSockets would add bidirectional complexity we don't need. Polling adds latency and load.

### Endpoints (simulator-only, under `/sim/*`)

| Endpoint | Purpose |
|---|---|
| `GET /` | The web UI (static bundle). |
| `GET /sim/events` | The SSE stream. It sends a full `state` snapshot on connect, then `patch`/`state` events on every change, plus `log` events for the request log and warnings. |
| `GET /sim/state` | The current `DeviceState` as JSON, for tests. |
| `GET /sim/frame.png?scale=N&t=<ms>` | The rendered frame as a PNG, optionally scaled up, at the current time or a given time. |
| `GET /sim/log` | Recent requests, responses and warnings as JSON. |
| `POST /sim/reset` | Reset the device to the profile defaults. |

### UI layout

- **Display:** a `<canvas>` that draws the 64×64 framebuffer as round "LEDs" with gaps and a dark bezel. It has zoom, a pixel-grid toggle and a coordinate read-out on hover, which helps when choosing `x`/`y` for text. It also offers "Download PNG" and pause/step for animations.
- **Device panel:** the profile name, active endpoints and ports, the active channel, brightness, rotation and mirror, screen on/off, the animation (ID, frame count, speed) and a list of text layers and items.
- **Request log:** one entry per request (command, parameters with `PicData` shortened, response, and the warnings from [ADR-0010](ADR-0010-validation-and-developer-feedback.md)), with filtering and "copy as curl".
- The UI is **read-only** by default. Manual device controls, such as a brightness slider, may be added later, but they must go through the same command dispatcher, never through a separate path.

### Consequences

- Good: zero-install, live and framework-free. Several tabs show identical output.
- Good: the PNG and JSON endpoints make the simulator usable in CI without a browser ([ADR-0011](ADR-0011-testing-strategy.md)).
- Bad: SSE connections count against a browser's per-host connection limit over HTTP/1.1. That rarely matters for a local tool.
- Bad: the UI port is the same as a device API port, so `/` and `/sim/*` are visible to device clients. That is harmless, because the real device does not use those paths.

# ADR-0004: Supported command set

- **Status:** Accepted
- **Date:** 2026-10-22

## Context and Problem Statement

The Pixoo local API has dozens of commands. Some draw content (animations, text, layouts). Some change device settings (brightness, rotation, channel). Some trigger built-in tools (timer, scoreboard, buzzer) or depend on things a simulator does not have (TF card, cloud gallery, microphone).

Implementing every command at full fidelity at once is neither possible nor useful. Which commands do we implement, and how faithfully?

## Decision Drivers

- Content developers mostly use the **drawing** commands. Those must be complete and accurate.
- Clients often **read settings back** (for example `Channel/GetAllConf`) and break if a command is missing. Every known command should get a well-formed reply.
- Commands we cannot simulate meaningfully should still be *visible* to the user.
- Unknown commands must never fail silently.

## Considered Options

1. **Tiered coverage**: full fidelity for drawing, state-level fidelity for settings, logged stubs for the rest
2. Drawing commands only, and errors for everything else
3. Every command at full fidelity before the first release

## Decision Outcome

Chosen option: **1, tiered coverage.** Every command name is registered in one **command registry**. Each entry records its parameter schema, its handler, its tier, and a short description that the web UI shows.

### Tier 1: Drawing (full fidelity, rendered)

| Command | Behaviour |
|---|---|
| `Draw/GetHttpGifId` | Returns `{"error_code":0,"PicId":<n>}`, the current animation ID counter. |
| `Draw/ResetHttpGifId` | Resets the counter and the animation buffers. |
| `Draw/SendHttpGif` | Accepts one frame of an animation: `PicNum` (frame count), `PicWidth` (16/32/64), `PicOffset` (frame index), `PicID`, `PicSpeed` (ms per frame) and `PicData` (base64 of raw RGB bytes, `PicWidth²×3`). Frames are buffered per `PicID`. Once all `PicNum` frames have arrived, the animation becomes the active content and the display switches to it. Smaller widths are scaled up to 64×64 with nearest-neighbour. |
| `Draw/SendHttpText` | Adds or replaces the text layer `TextId` drawn over the animation: `x`, `y`, `dir` (scroll direction), `font`, `TextWidth`, `speed`, `TextString`, `color` (`#RRGGBB`) and `align`. |
| `Draw/ClearHttpText` | Removes all text layers. |
| `Draw/SendHttpItemList` | Adds or replaces a list of items (`ItemList[]`). Each item has `TextId`, `type`, `x`, `y`, `dir`, `font`, `TextWidth`, `Textheight`, `speed`, `align`, `color`, `TextString`, `update_time` and so on. Dynamic types (time, date, temperature and similar) are rendered from the simulator clock and simulated weather values. Type `22` is literal text. Type `23` (net text, fetched from a URL) is fetched by the simulator. |
| `Draw/CommandList` | Runs each entry of `CommandList[]` in order through the same dispatcher, then returns a single `{"error_code":0}`. |
| `Draw/UseHTTPCommandSource` | Fetches a JSON command list from `CommandUrl` and runs it as with `Draw/CommandList`. |

### Tier 2: Device state (stored, reflected in rendering or read-backs)

- **Channel:** `Channel/SetIndex` / `Channel/GetIndex` (0 Faces, 1 Cloud, 2 Visualizer, 3 Custom), `Channel/SetClockSelectId`, `Channel/GetClockInfo`, `Channel/SetCustomPageIndex`, `Channel/SetEqPosition`, `Channel/CloudIndex`, `Channel/SetBrightness`, `Channel/GetAllConf`, `Channel/OnOffScreen`.
- **Device:** `Device/SetScreenRotationAngle`, `Device/SetMirrorMode`, `Device/SetTime24Flag`, `Device/SetDisTempMode`, `Device/SetHighLightMode`, `Device/SetWhiteBalance`, `Device/SetUTC`, `Device/GetDeviceTime`, `Device/GetWeatherInfo` (returns simulated values).
- **System:** `Sys/TimeZone`, `Sys/LogAndLat`.

Settings that change the picture (brightness, screen on/off, rotation, mirror, white balance, channel) affect the renderer ([ADR-0007](ADR-0007-state-model-and-rendering-pipeline.md)). All settings show up in `Channel/GetAllConf` and the other read commands, with the field names the device uses.

### Tier 3: Acknowledged stubs (logged, shown as overlays or badges where useful)

`Tools/SetTimer`, `Tools/SetStopWatch`, `Tools/SetScoreBoard`, `Tools/SetNoiseStatus`, `Device/PlayBuzzer`, `Device/PlayTFGif`, `Draw/SendRemote`, `Device/SysReboot`, and similar commands.

These validate their parameters, return `{"error_code":0}`, are recorded in the request log, and where practical show an indicator in the UI (for example "Buzzer: 3 × 500 ms"). Later versions may move some of them up to Tier 2 (a rendered scoreboard or timer, for instance).

### Unknown commands

These are logged with a warning and get an error response ([ADR-0003](ADR-0003-wire-compatible-http-api.md)). The web UI points them out, because they usually mean a typo in the client.

### Consequences

- Good: the most valuable surface, custom drawing, is complete from the start.
- Good: clients that read settings back do not break.
- Good: one registry drives dispatching, validation ([ADR-0010](ADR-0010-validation-and-developer-feedback.md)) and documentation.
- Bad: Tier 3 commands "succeed" without visible effect. The UI must make that clear so nobody assumes the content was verified.

## More Information

### Open questions to verify on hardware

- The full list of `SendHttpItemList` `type` values and how each is rendered.
- How text layers behave when the animation is replaced: are they kept or cleared?
- The maximum `PicNum` and total payload the device accepts.
- The exact field set and defaults of `Channel/GetAllConf` on current firmware.

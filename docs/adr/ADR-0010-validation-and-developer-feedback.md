# ADR-0010: Validation and developer feedback

- **Status:** Accepted
- **Date:** 2026-10-22

## Context and Problem Statement

The real device is very lenient. Many incorrect requests return `{"error_code": 0}` and then show nothing, garbage, or stale content. Some well-known community pitfalls:

- `PicData` whose decoded length is not `PicWidth × PicWidth × 3`.
- Reusing or decreasing `PicID` instead of taking it from `Draw/GetHttpGifId` and incrementing it.
- Sending text (`Draw/SendHttpText`) before any HTTP animation exists to draw it over.
- Uploading very many animations without `Draw/ResetHttpGifId`. Users report the device freezing once the counter gets high.
- Values out of range: brightness outside 0–100, unsupported `PicWidth`, `PicOffset ≥ PicNum`, malformed `color`, unknown font IDs.
- Sending too many frames, or too much data, in one animation.

If the simulator copied the device's silence, it would hide exactly the bugs it exists to catch. How should it report problems without breaking compatibility ([ADR-0003](ADR-0003-wire-compatible-http-api.md))?

## Decision Drivers

- Responses stay wire-compatible by default, so clients behave as they would against the device.
- Problems must still be obvious to the developer.
- Optional strictness for CI, where a warning should fail the build.

## Considered Options

1. **Device-compatible responses plus out-of-band warnings, with an opt-in strict mode**
2. Return errors for every problem (always strict)
3. Copy the device's behaviour exactly, with no extra feedback

## Decision Outcome

Chosen option: **1.**

- Every command parameter is checked against the schema in the command registry ([ADR-0004](ADR-0004-supported-command-set.md)) and against the semantic rules above. The limits come from the active device profile ([ADR-0005](ADR-0005-device-profiles.md)).
- Findings have a severity:
  - **error:** the device would certainly not display what was intended, for example a wrong `PicData` length.
  - **warning:** likely a mistake, or a known hardware risk, for example a non-increasing `PicID` or a PicID counter above the profile threshold.
  - **info:** worth knowing, for example "text is wider than TextWidth and will scroll" or "Tier 3 command acknowledged but not simulated".
- **Default mode:** the device-compatible response is returned unchanged. Findings go to the request log, the SSE `log` events and the web UI ([ADR-0009](ADR-0009-web-ui-and-live-updates.md)). Optionally they also go into an extra response header, `X-Pixel-Sim-Warnings`, which clients ignore.
- **Strict mode** (`--strict`): any finding of severity *error*, or *warning* with `--strict=warning`, makes the response carry a non-zero `error_code` with a readable message. The command is not applied.
- Each rule has a stable ID (for example `PIC_DATA_LENGTH`, `PIC_ID_NOT_INCREASING`) so it can be documented, filtered and turned off (`--ignore PIC_ID_NOT_INCREASING`).
- The simulator **never** reproduces hardware failures (freezing, reboots). It warns about them instead ([ADR-0002](ADR-0002-simulator-vision-and-scope.md)).

### Consequences

- Good: clients behave as they would against the device, while developers see every problem.
- Good: strict mode turns the simulator into a contract checker for CI ([ADR-0011](ADR-0011-testing-strategy.md)).
- Bad: some rules encode community knowledge rather than official specification, and may be wrong for some firmware. Rule IDs and ignore flags limit the damage, and each rule documents its source.

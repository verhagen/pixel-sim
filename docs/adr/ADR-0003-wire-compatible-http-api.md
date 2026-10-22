# ADR-0003: Wire-compatible device HTTP API

- **Status:** Proposed (to be confirmed against real Pixoo-64 / Pixoo-64 II hardware)
- **Date:** 2026-10-22

## Context and Problem Statement

The main promise of the simulator ([ADR-0002](ADR-0002-simulator-vision-and-scope.md)) is that **existing clients work unchanged**. Clients reach the real device in two ways:

- **Legacy endpoint (documented):** `POST http://<device-ip>:80/post`. The body is a JSON object whose `Command` field names the operation, with the parameters as sibling fields:

  ```json
  { "Command": "Channel/SetBrightness", "Brightness": 80 }
  ```

  The response is JSON that always contains `error_code`. A value of `0` means success, and some commands add fields of their own:

  ```json
  { "error_code": 0, "Brightness": 80, "RotationFlag": 0 }
  ```

- **Newer endpoint (reported by the community, not officially documented):** later hardware or firmware revisions serve `/divoom_api` on **port 9000**, and the legacy `/post` endpoint is gone or refuses connections on port 80. Client libraries have started probing both. Reports conflict on whether a Pixoo-64 II uses the legacy or the new revision.

Clients also differ in what they send. Many omit `Content-Type: application/json` or send `text/plain`. Some send compact JSON, some pretty-printed. Browser-based clients need CORS.

How strictly do we reproduce this interface, and which variants do we support?

## Decision Drivers

- Unmodified clients must work: same paths, same JSON shape, same success semantics.
- Both known endpoint generations must be supported, because clients in the wild target both.
- The device API namespace must stay free of simulator-only features.
- Be tolerant on input, as the device appears to be.

## Considered Options

1. **Byte-compatible emulation of both endpoint generations, configurable per device profile**
2. Only the documented `/post` endpoint
3. A new, "cleaner" REST API (e.g. `PUT /brightness`) with an adapter for Pixoo clients

## Decision Outcome

Chosen option: **1.**

- The simulator serves the **device API** on configurable listeners:
  - `POST /post`, by default on port **80**, configurable with `--port` because binding to 80 usually needs privileges.
  - `POST /divoom_api`, by default on port **9000**, enabled per device profile ([ADR-0005](ADR-0005-device-profiles.md)) and configurable with `--api-port`.
  - Both paths share the same dispatcher, so a command behaves the same whichever endpoint receives it.
- **Request parsing:** the body is read as UTF-8 and parsed as JSON **whatever the `Content-Type`**. `Command` is matched case-sensitively, as the device does, and unknown extra fields are ignored.
- **Response format:** `Content-Type: application/json`, with HTTP status **200** even for application errors, and `error_code: 0` on success. Read commands return the fields the device returns ([ADR-0004](ADR-0004-supported-command-set.md)).
- **Errors:** invalid JSON, a missing `Command`, or an unknown `Command` return HTTP 200 with a non-zero `error_code`, modelled on the real device's error strings as far as they are known. Where the real response is unknown, the shape is a documented assumption.
- **CORS:** permissive (`Access-Control-Allow-Origin: *`, answers `OPTIONS` preflights), so browser-based tools can use the simulator. This is a deliberate extension. The real device's CORS behaviour is not reproduced.
- **Simulator-only endpoints** live under a reserved prefix, `/sim/*` (state, rendered frames, the event stream, control), and the web UI is served at `/`. The real device does not use these paths, so they cannot collide with device commands ([ADR-0009](ADR-0009-web-ui-and-live-updates.md), [ADR-0010](ADR-0010-validation-and-developer-feedback.md)).
- **No authentication**, matching the device.

### Consequences

- Good: clients only need to change the host and port.
- Good: one dispatcher covers both endpoint generations, so they cannot drift apart.
- Bad: because both generations run in one process, a client probing for the "right" endpoint may find both, which a real device would never offer. Profiles can turn either endpoint off to reproduce a specific revision.
- Bad: exact error strings and edge-case responses are partly guesses until they are checked on hardware. This ADR stays *Proposed* until then.

## More Information

### Open questions to verify on hardware

- Exact `error_code` values or strings for invalid JSON, a missing `Command` and an unknown `Command`.
- Whether `/divoom_api` on port 9000 uses the same JSON envelope as `/post`, and whether it also accepts GET.
- Which endpoint generation each Pixoo-64 II firmware version exposes.
- Whether the device sends any CORS headers.

### Sources

- Divoom local API documentation: <http://doc.divoom-gz.com/web/#/12?page_id=196>
- Community reports about `/divoom_api` on port 9000 and legacy/new revisions: Homey community thread "[APP][Pro] Divoom Pixoo64" and the Home Assistant community thread "Pixoo REST — Home Assistant add-on + native integration for Divoom Pixoo64".

# ADR-0005: Device profiles for Pixoo-64 and Pixoo-64 II

- **Status:** Proposed (Pixoo-64 II specifics to be confirmed on hardware)
- **Date:** 2026-10-22

## Context and Problem Statement

The simulator has to cover the **Pixoo-64** and the **Pixoo-64 II**. As far as we know, both are 64×64 displays that accept the same command set ([ADR-0004](ADR-0004-supported-command-set.md)). Some details may still differ between models or firmware revisions:

- Which HTTP endpoint is exposed: legacy `/post` on port 80, or `/divoom_api` on port 9000 ([ADR-0003](ADR-0003-wire-compatible-http-api.md)).
- Limits, such as the maximum number of animation frames or how far the animation ID counter can go before the device misbehaves.
- Defaults reported by read commands (device name, default brightness, clock ID).
- Model-specific features we have not discovered yet.

How do we support several device models without duplicating the implementation?

## Decision Drivers

- One code path. Model differences should be *data*, not forks of the code.
- Adding a model or firmware revision later should be cheap.
- Users should be able to reproduce the exact revision they own.
- Running several simulated devices at once (for example to test a multi-device setup) should be possible.

## Considered Options

1. **Declarative device profiles** selected at start-up
2. A separate simulator implementation per model
3. A single generic "Pixoo" with no model distinction

## Decision Outcome

Chosen option: **1, declarative device profiles.**

A `DeviceProfile` is a typed, read-only object. Conceptually:

```ts
interface DeviceProfile {
  id: string;                  // "pixoo-64", "pixoo-64-ii"
  displayName: string;         // shown in the UI and in discovery responses
  resolution: { width: 64; height: 64 };
  endpoints: {
    legacyPost: { enabled: boolean; defaultPort: number };   // POST /post
    divoomApi:  { enabled: boolean; defaultPort: number };   // POST /divoom_api
  };
  limits: {
    allowedPicWidths: number[];      // [16, 32, 64]
    maxFrames: number;               // warn above this
    picIdWarnThreshold: number;      // known counter lock-up quirk
  };
  defaults: Partial<DeviceSettings>; // brightness, channel, clock id, ...
  features: Record<string, boolean>; // model-specific switches
}
```

Shipped profiles:

| Profile | Endpoints enabled by default | Notes |
|---|---|---|
| `pixoo-64` | `/post` (port 80) | Reference behaviour, matching the documented API. |
| `pixoo-64-ii` | `/post` (port 80) **and** `/divoom_api` (port 9000) | Treated as API-identical to `pixoo-64` until differences are confirmed. Both endpoints are on, because reports conflict. |

Selection and overrides:

- CLI: `pixel-sim --device pixoo-64-ii [--port 8080] [--api-port 9000] [--no-legacy] [--no-divoom-api]`.
- The same options can come from environment variables, for container use.
- **One simulated device per process.** To simulate several devices, start several processes on different ports. That keeps the state model simple ([ADR-0007](ADR-0007-state-model-and-rendering-pipeline.md)).

### Consequences

- Good: a model difference we discover later becomes a profile change, not a code change.
- Good: users can turn endpoints off to reproduce exactly what their client will meet.
- Bad: until we have tested on hardware, `pixoo-64-ii` is mostly an alias. The README must say so.
- Bad: one device per process means simulating many devices takes several processes. That is acceptable for the expected use.

## More Information

### Open questions to verify on hardware

- Which endpoint(s) and port(s) a Pixoo-64 II exposes on each firmware version.
- Whether the Pixoo-64 II adds or changes commands or response fields.
- The real limits for frame count and the PicID counter on both models.

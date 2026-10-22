# ADR-0008: Bitmap fonts for device text

- **Status:** Accepted
- **Date:** 2026-10-22

## Context and Problem Statement

`Draw/SendHttpText` and `Draw/SendHttpItemList` draw text on a 64×64 grid. Each layer chooses a `font` by numeric ID and sets `align`, `TextWidth`, scroll direction (`dir`) and `speed`. On the real device these are small pixel fonts. Every lit pixel is a single LED, so whether text *fits* and *reads well* depends entirely on the exact glyph shapes.

How does the simulator rasterise device text?

## Decision Drivers

- Predict accurately whether text fits in `TextWidth` and how it scrolls.
- The same output in every browser and on the server ([ADR-0007](ADR-0007-state-model-and-rendering-pipeline.md)).
- No anti-aliasing. Every pixel is fully on or fully off.
- Fonts must be shippable without licensing problems.

## Considered Options

1. **Embedded bitmap fonts** stored as data in the shared renderer module
2. Browser canvas `fillText` with a pixel-style web font, thresholded to on/off
3. Server-side font rasterisation of TTF fonts, for example with FreeType bindings

## Decision Outcome

Chosen option: **1, embedded bitmap fonts.**

- The fonts are stored as compact glyph tables (per-glyph width, height and bitmap rows) in a TypeScript module that the server and the browser both use.
- At least two fonts ship at first: a **small font** (about 3×5) and a **standard font** (about 5×7). Both cover printable ASCII, plus a fallback glyph for unsupported characters.
- A per-profile `fontMap` maps device `font` IDs to embedded fonts ([ADR-0005](ADR-0005-device-profiles.md)). Unknown IDs fall back to the standard font and raise a validation warning ([ADR-0010](ADR-0010-validation-and-developer-feedback.md)).
- Layout rules:
  - The text box is `TextWidth` pixels wide, starting at `x`, `y`, and pixels outside the box are clipped.
  - `align`: 1 left, 2 centre, 3 right, applied when the text fits.
  - Text wider than the box scrolls in direction `dir` (0 left, 1 right), moving one pixel every `speed` ms and wrapping round.
  - `color` is parsed from `#RRGGBB`.
- Fonts must have an open licence, such as public domain, CC0 or OFL bitmap fonts, or be drawn for this project.

### Consequences

- Good: rendering is deterministic, pixel-exact and identical everywhere, with no font-loading race conditions.
- Good: width measurement is exact, so warnings like "text overflows and will scroll" are reliable.
- Bad: the embedded fonts will not match Divoom's built-in fonts glyph for glyph. Font fidelity is a known limitation until real device fonts can be documented or approximated more closely.
- Bad: non-Latin scripts (the device supports some) are out of scope at first.

## More Information

### Open questions to verify on hardware

- How many font IDs the device has, and the size and shape of each font.
- The exact `align` and `dir` codes, and the units of `speed`.
- Behaviour when text fits inside `TextWidth`: is it static or still scrolling?

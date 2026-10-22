# ADR-0011: Testing strategy

- **Status:** Accepted
- **Date:** 2026-10-22

## Context and Problem Statement

The simulator is only useful if it behaves like the real device. Its own correctness has two sides:

1. **Internal correctness:** handlers, the state model, the renderer and validation do what the ADRs say.
2. **Fidelity:** what the ADRs say matches the actual Pixoo-64 and Pixoo-64 II. Several ADRs list open questions to verify on hardware.

How do we test both, and how do users test their own content with the simulator?

## Decision Drivers

- Fast, deterministic tests that need no hardware and no real-time waits.
- A way to capture real device behaviour once and keep checking against it.
- Users should be able to use the simulator in their own CI.

## Considered Options

1. **Layered tests**: unit tests, golden frames, contract fixtures, plus hardware verification notes
2. Unit tests only
3. End-to-end browser tests only

## Decision Outcome

Chosen option: **1.** The test runner is Vitest ([ADR-0006](ADR-0006-technology-stack.md)).

- **Unit tests** for each command handler (state before, then state after, plus the response) and for each validation rule (input, then findings).
- **Renderer golden tests:** the pure renderer ([ADR-0007](ADR-0007-state-model-and-rendering-pipeline.md)) is called with fixed state and explicit timestamps, and the output is compared with committed golden PNGs. Covered cases include animation frame selection, PicWidth scaling, text alignment and scrolling at given times, rotation, mirroring, brightness and white balance. A failing test writes a diff image.
- **HTTP contract tests:** the real server is started on an ephemeral port. Request/response fixtures in JSON are replayed against both `/post` and `/divoom_api`, and responses are compared field by field. Wherever possible the fixtures are **captured from real devices**, and each records the model and firmware version. Fixtures based on assumptions are marked as such.
- **Compatibility smoke tests** (optional, in CI): run popular community client libraries against the simulator, to catch protocol mismatches.
- **Hardware verification log:** the open questions from the ADRs are tracked. Whenever one is answered on a real device, a fixture is added and the ADR concerned is updated. Its status moves from *Proposed* to *Accepted*, or a superseding ADR is written.
- **For users:** the documentation explains how to use the simulator in CI. Start it in `--strict` mode, run the client, then check `/sim/state`, `/sim/log` or `/sim/frame.png` against expectations.

### Consequences

- Good: the core logic is fast to test and fully deterministic.
- Good: fidelity improves step by step and stays measurable through recorded fixtures.
- Bad: golden PNGs have to be regenerated on purpose when rendering changes. Reviewers must check the diffs.
- Bad: fidelity depends on someone with access to real hardware contributing fixtures.

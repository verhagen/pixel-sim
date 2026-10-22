# ADR-0002: Simulator vision and scope

- **Status:** Accepted
- **Date:** 2026-10-22

## Context and Problem Statement

The Divoom **Pixoo-64** and **Pixoo-64 II** are 64×64 RGB LED pixel displays. Besides the mobile app, they expose a local **HTTP API** on the LAN. With it, scripts, home-automation systems (Home Assistant, Homey, Node-RED) and community libraries can push animations, text and dashboard-like layouts to the display.

Building content for these displays today is slow and awkward:

- Every change means sending it to physical hardware and looking at the device.
- You need to own a device and be on the same network as it.
- The device gives almost no feedback. A malformed request often "succeeds" (`{"error_code": 0}`) while showing nothing, garbage, or the previous content.
- Some firmware quirks are known to the community but not documented. For example, the animation ID counter can lock the display after many uploads.
- Automated tests (CI) for code that targets a Pixoo are effectively impossible.

How can people check that their idea for displaying certain content works as intended **before** pushing it to a real device?

## Decision Drivers

- **Drop-in compatibility:** existing clients and libraries should need no code changes, only a different IP address or port.
- **Visual fidelity:** what the simulator shows should match the real 64×64 display closely enough to judge layout, colours, animation timing and text scrolling.
- **Better feedback than the device:** the simulator should explain what is wrong, not just reproduce the silence.
- **Easy to run:** one command, local, no account, no cloud.
- **Usable in automation:** state and rendered frames should be readable by tests.

## Considered Options

1. **An HTTP-compatible simulator that renders in a web page** (this project)
2. A client-side library or preview tool for one language, such as a Python preview window
3. A mock server that only records requests and does not render anything
4. Emulating the device firmware

## Decision Outcome

Chosen option: **1, a standalone simulator process.** It exposes the **same local HTTP API** as the Pixoo-64 and Pixoo-64 II, keeps the device state the way the device would, and **renders the display live in a web page**.

Because the integration point is the HTTP wire protocol, the simulator works with any client in any language (option 2 does not). It gives visual results (option 3 does not). And it is far cheaper and more transparent than firmware emulation (option 4).

### Goals

- Accept the device's local HTTP API at the same paths, with the same request and response JSON shapes ([ADR-0003](ADR-0003-wire-compatible-http-api.md)), covering the commands that matter for content ([ADR-0004](ADR-0004-supported-command-set.md)).
- Simulate both the Pixoo-64 and the Pixoo-64 II through device profiles ([ADR-0005](ADR-0005-device-profiles.md)).
- Render a faithful 64×64 picture: animations, text and item-list overlays, brightness, rotation and mirror ([ADR-0007](ADR-0007-state-model-and-rendering-pipeline.md), [ADR-0008](ADR-0008-bitmap-fonts-for-text.md)).
- Show the display, the device state and a request log in a browser ([ADR-0009](ADR-0009-web-ui-and-live-updates.md)).
- Warn about requests that would misbehave on real hardware ([ADR-0010](ADR-0010-validation-and-developer-feedback.md)).
- Let tests read the state and the frames ([ADR-0009](ADR-0009-web-ui-and-live-updates.md), [ADR-0011](ADR-0011-testing-strategy.md)).

### Non-goals

- **No Divoom cloud APIs** (`app.divoom-gz.com` and similar): no gallery and no account features. A local stub for LAN device discovery may come later.
- **No Bluetooth or mobile-app protocol.** HTTP only.
- **No pixel-perfect replicas of built-in content.** Clock faces, cloud channel and audio visualizer are shown as clearly labelled placeholders.
- **No real audio**, so no buzzer sound and no microphone-driven visualizer.
- **No TF/SD-card file system.** Commands that play files from the card are acknowledged and logged only.
- **No reproduction of hardware failures** such as the display freezing. The simulator *warns* about them instead of freezing.

### Target users

- Developers of Pixoo client libraries and integrations.
- Home-automation users designing dashboards or notifications.
- Anyone who wants to try an idea without owning, or without disturbing, a physical device.

### Success criteria

- A client written for a real Pixoo runs unchanged against the simulator by pointing it at the simulator's host and port.
- Content shown in the simulator looks the same on a real device, apart from documented limitations.
- Common mistakes, such as a wrong `PicData` length or a non-increasing `PicID`, produce a clear warning.

### Consequences

- Good: works with every language and library that already speaks the Pixoo HTTP API.
- Good: helps with design, debugging and CI alike.
- Bad: the device API is only partly documented, so fidelity depends on what we observe on hardware. Assumptions have to be written down and verified ([ADR-0011](ADR-0011-testing-strategy.md)).
- Bad: built-in content (clock faces, visualizer) is not reproduced, so the simulator is mainly useful for **custom HTTP-driven content**.

## More Information

- Divoom local API documentation (old official location): <http://doc.divoom-gz.com/web/#/12?page_id=196>
- Divoom, "Pixoo-64 API Beginner Guide": <https://divoom.com/blogs/app-guide/pixoo-64-api-beginner-guide>

# ADR-0006: Technology stack (Node.js + TypeScript)

- **Status:** Accepted
- **Date:** 2026-10-22

## Context and Problem Statement

The simulator has two halves. One is an **HTTP server** that emulates the device API and keeps the device state. The other is a **web page** that shows the display. Both need the same knowledge: the device state model, and above all how that state turns into 64×64 pixels ([ADR-0007](ADR-0007-state-model-and-rendering-pipeline.md)). Which language and runtime do we use?

## Decision Drivers

- Share the rendering and state logic between server and browser, so it is written once and cannot diverge.
- Model the command parameter schemas explicitly ([ADR-0004](ADR-0004-supported-command-set.md), [ADR-0010](ADR-0010-validation-and-developer-feedback.md)).
- Easy installation for users: `npx`, a global npm install, or a container.
- Few runtime dependencies, so the attack surface and maintenance stay small.
- A mainstream ecosystem, so contributors find it familiar.

## Considered Options

1. **Node.js (LTS) + TypeScript**
2. Node.js + plain JavaScript (no build step)
3. Python (FastAPI or aiohttp) for the server, with a separate JavaScript front-end
4. Go or Rust single binary, with a separate JavaScript front-end

## Decision Outcome

Chosen option: **1, Node.js LTS + TypeScript.**

- **Runtime:** the current Node.js LTS. The minimum supported version is pinned in `package.json` `engines`.
- **Language:** TypeScript in `strict` mode for all code. Shared modules (state model, command schemas, renderer, fonts) are environment-neutral ES modules. They do not use Node-only or DOM-only APIs, so the server and the browser bundle can both import them.
- **HTTP server:** the built-in `node:http`, with a small router of our own. The device API is one POST endpoint plus a handful of `/sim/*` routes, so a framework adds little.
- **Front-end:** plain TypeScript with a `<canvas>`, bundled with **Vite** (or esbuild). No UI framework at first. One can be added if the UI grows.
- **PNG encoding** for `/sim/frame.png`: a small, well-maintained dependency (for example `pngjs`), or a minimal encoder using `node:zlib`.
- **Testing:** **Vitest** ([ADR-0011](ADR-0011-testing-strategy.md)).
- **Tooling:** ESLint and Prettier, with npm scripts `dev`, `build`, `test`, `lint` and `start`.
- **Distribution:** an npm package with a `pixel-sim` binary, plus an optional Dockerfile.

### Consequences

- Good: the renderer is written once and runs both in the browser (live view) and on the server (PNG snapshots, tests), so the two always match.
- Good: TypeScript types document the command parameters and the state shape.
- Good: Node's standard library covers HTTP, zlib and fetch, so few dependencies are needed.
- Bad: a build step (tsc/Vite) is needed, compared with plain JavaScript.
- Bad: Python-centric Pixoo users (many existing libraries are Python) get a Node tool. This is acceptable, because users talk to the simulator over HTTP, not through its source code.

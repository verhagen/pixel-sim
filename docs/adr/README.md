# Architecture Decision Records

This directory holds the Architecture Decision Records (ADRs) for **pixel-sim**, a simulator for the Divoom Pixoo-64 and Pixoo-64 II that speaks the devices' local HTTP API and renders the display in a web page.

Start with [ADR-0002](ADR-0002-simulator-vision-and-scope.md) for the overall idea.

## Index

| ADR | Title | Status |
|---|---|---|
| [ADR-0001](ADR-0001-record-architecture-decisions.md) | Record architecture decisions | Accepted |
| [ADR-0002](ADR-0002-simulator-vision-and-scope.md) | Simulator vision and scope | Accepted |
| [ADR-0003](ADR-0003-wire-compatible-http-api.md) | Wire-compatible device HTTP API | Proposed |
| [ADR-0004](ADR-0004-supported-command-set.md) | Supported command set | Accepted |
| [ADR-0005](ADR-0005-device-profiles.md) | Device profiles for Pixoo-64 and Pixoo-64 II | Proposed |
| [ADR-0006](ADR-0006-technology-stack.md) | Technology stack (Node.js + TypeScript) | Accepted |
| [ADR-0007](ADR-0007-state-model-and-rendering-pipeline.md) | State model and rendering pipeline | Accepted |
| [ADR-0008](ADR-0008-bitmap-fonts-for-text.md) | Bitmap fonts for device text | Accepted |
| [ADR-0009](ADR-0009-web-ui-and-live-updates.md) | Web UI and live updates | Accepted |
| [ADR-0010](ADR-0010-validation-and-developer-feedback.md) | Validation and developer feedback | Accepted |
| [ADR-0011](ADR-0011-testing-strategy.md) | Testing strategy | Accepted |

## Conventions

- Format: [MADR](https://adr.github.io/madr/) (see [ADR-0001](ADR-0001-record-architecture-decisions.md)).
- File name: `ADR-####-title.md`. `####` is the next free zero-padded 4-digit number, and `title` is kebab-case.
- Status: `Proposed`, `Accepted`, `Deprecated` or `Superseded by ADR-####`. Never delete or renumber an ADR. Supersede it instead.
- When you add an ADR, add it to the index above in the same pull request.

## Template

```markdown
# ADR-####: <Title>

- **Status:** Proposed
- **Date:** YYYY-MM-DD

## Context and Problem Statement

<What is the situation and which question needs an answer?>

## Decision Drivers

- <driver>

## Considered Options

1. <option>
2. <option>

## Decision Outcome

Chosen option: **<option>**, because <justification>.

### Consequences

- Good: <...>
- Bad: <...>

## More Information

<Links, open questions, verification notes.>
```

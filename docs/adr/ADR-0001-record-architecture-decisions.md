# ADR-0001: Record architecture decisions

- **Status:** Accepted
- **Date:** 2026-10-22

## Context and Problem Statement

pixel-sim is a new project that has no code yet. Before anything is built we need to agree on what the simulator is, what it is not, and how it will be built. Those decisions should stay findable and reviewable, and they should be easy to revise as we learn more about the real devices. Where should we record them, and in what form?

## Decision Drivers

- Decisions should live next to the code and be versioned with it.
- A low barrier: plain Markdown, no tooling required.
- Every decision should show the alternatives that were considered and why they lost.
- Superseding a decision later should be explicit, not a silent edit.

## Considered Options

1. Architecture Decision Records in the [MADR](https://adr.github.io/madr/) format, stored in the repository
2. Architecture Decision Records in the classic Nygard format
3. A single design document (e.g. `DESIGN.md`)
4. A wiki or an external document

## Decision Outcome

Chosen option: **1, MADR records in `docs/adr/`**, because MADR explicitly captures decision drivers and considered options. That matters here because many choices depend on the behaviour of real hardware that we still need to verify.

Conventions:

- Location: `docs/adr/`.
- File name: `ADR-####-title.md`. `####` is a zero-padded 4-digit sequence number and `title` is a short kebab-case summary, for example `ADR-0007-state-model-and-rendering-pipeline.md`.
- Numbers are never reused. A record that is replaced gets the status `Superseded by ADR-####`, and its file stays in place.
- Status values: `Proposed`, `Accepted`, `Deprecated`, `Superseded by ADR-####`.
- [`docs/adr/README.md`](README.md) holds the index and a copy of the template.

### Consequences

- Good: decisions are reviewed in pull requests like any other change.
- Good: newcomers can read the reasoning in order, starting with the vision ([ADR-0002](ADR-0002-simulator-vision-and-scope.md)).
- Bad: the records have to be kept in step with reality. A stale ADR is worse than none, so changing a decision means updating or superseding its ADR in the same pull request.

## More Information

- MADR: <https://adr.github.io/madr/>
- Michael Nygard, "Documenting Architecture Decisions" (2011)

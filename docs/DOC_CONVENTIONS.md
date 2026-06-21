# Documentation conventions

How this repository is documented, so the build stays reproducible and explainable —
and so anyone (including future-us) can understand *what we did and why*. New project
repos should follow the same shape.

## Layout

| Path | Purpose |
|---|---|
| `README.md` | One-line "what + why" first; then what-it-does, stack, layout, status, license |
| `docs/project-plan.md` | Phased roadmap + live status table (the centrepiece) |
| `docs/architecture.md` | Design principles, trade-offs, hardware/topology, data sources |
| `docs/inventory.md` | Baseline facts the build assumes |
| `docs/decisions/NNNN-*.md` | ADRs — the durable "why" for each significant choice |
| `docs/phases/*.md` | Per-phase build logs (added as each phase is executed) |
| `CHANGELOG.md` | Keep a Changelog + SemVer; updated every phase |
| `packages/` | Version-controlled HA YAML config |
| `prompts/` | AI Task prompt templates |

## Project plan format

Each phase documents: **Objective · What we build · Prerequisites · Deliverables ·
Why (rationale) · Exit criteria**. A status table at the top tracks progress:
☐ not started · ◐ in progress · ☑ done. Keep an **Open decisions** section for choices
deferred or awaiting input.

## Decision records (ADRs)

One file per significant choice, numbered. Each has: **Status, Date, Context, Decision,
Consequences**. Write these when a choice would otherwise be invisible later — they are
the answer to "why did we do it this way?".

## Changelog & releases

- Update `CHANGELOG.md` (`[Unreleased]`) with **every** change; phases map loosely to minor
  versions (Phase 1 → v0.1.0).
- Releases follow [release-process.md](release-process.md) and happen **only when explicitly
  asked** — pushing the repo is not a release.

## Principles

- The **repo is the source of truth for intent + rationale**.
- Version-controlled config lives in the repo as code (YAML/etc.); never hand-edit opaque
  state stores (e.g. HA `.storage/`) — see [ADR-0002](decisions/0002-repository-as-source-of-truth.md).
- Never commit secrets, credentials, or runtime state.

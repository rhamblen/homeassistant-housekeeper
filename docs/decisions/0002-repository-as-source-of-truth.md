# ADR-0002 — Repository documents the build; YAML packages where version control matters

- **Status:** Accepted
- **Date:** 2026-06-21

## Context

We want to publish this work on GitHub and "come back at a future date knowing what we
did and why." But many HA helpers/automations are created through the UI/MCP and stored
in `.storage/` — which is **not** human-readable or git-trackable. HA best practice also
discourages hand-editing `.storage` or telling users to paste raw YAML for things the UI
manages.

## Decision

1. **The repo is the source of truth for *intent and rationale*** — plans, decisions,
   prompt templates, helper/automation specifications, and inventory.
2. **For anything we want versioned and reproducible**, prefer **HA YAML packages**
   (`packages/`) — they are git-trackable and survive a rebuild. Helpers that only exist
   sensibly as UI objects are *documented* here (definition + why) even if created via MCP.
3. **Helpers/automations are created through proper HA mechanisms** (MCP tools / UI /
   YAML packages) — never by editing `.storage` by hand.
4. **Every phase** updates `project-plan.md` status and `CHANGELOG.md`.

## Consequences

- Future-us can rebuild or understand the system from the repo alone.
- A small amount of duplication (UI object + its documented spec) is accepted as the price
  of explainability.
- Release/versioning follows [release-process.md](../release-process.md).

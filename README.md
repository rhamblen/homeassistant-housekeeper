# AI Home Manager for Home Assistant

Turning a large Home Assistant install into a proactive *home manager* using a
**local LLM** (Ollama on an RTX 3090) — not just a voice toy. It watches consumption,
explains what's happening, and flags problems before they bite, while keeping data on
the LAN.

## The idea in one line

**Home Assistant measures and remembers (the analyst); the local LLM explains and
advises (the narrator).** See [docs/architecture.md](docs/architecture.md).

## What it does (by phase)

1. **Capture** per-device energy history with HA helpers.
2. **Narrate** it — a morning briefing, per-wash cost, daily energy review.
3. **Watch** for anomalies — leaks, offline devices, things left on — quietly, only
   speaking when it matters.
4. **Learn** what each device/room draws (fingerprinting), mostly passively.
5. **See** — vision-model camera alerts ("courier left a parcel", not "motion detected").
6. **Converse & optimise** — natural-language control and "good time to run a wash" advice.

Full roadmap with goals, deliverables, and rationale: **[docs/project-plan.md](docs/project-plan.md)**.

## Stack

- Home Assistant `192.168.1.64` (2026.7).
- Ollama `192.168.1.32:11434` on Unraid (RTX 3090 24 GB + RTX 3060 12 GB).
- Models: `qwen2.5:14b` (brain), `qwen2.5vl:7b` (vision).
- Real-time power from **myenergi**; cost/tariff from **Octopus**.

## Repository layout

```
docs/
  project-plan.md      # phased roadmap (the centrepiece)
  architecture.md      # design, hardware, network, data sources
  inventory.md         # what's already metered (baseline)
  release-process.md   # how we version & publish
  decisions/           # ADRs — the "why" that must survive
  phases/              # working notes per phase, added as we build
packages/              # HA YAML packages (version-controlled config)
prompts/               # AI Task prompt templates
CHANGELOG.md
```

## Status

Phase 0 (foundations). See the status table in
[docs/project-plan.md](docs/project-plan.md). Built and documented with
[Claude Code](https://claude.com/claude-code).

## Why publish this

Two reasons: so the build is reproducible and explainable to future-us, and because the
analyst/narrator pattern on top of a big real-world HA install may help others doing the
same. License: [MIT](LICENSE).

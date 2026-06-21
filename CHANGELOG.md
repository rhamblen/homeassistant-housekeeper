# Changelog

All notable changes to this project are documented here.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning: [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Repository scaffold: README, MIT license, changelog, `.gitignore`.
- `docs/project-plan.md` — 8-phase roadmap (0–7) with goals, deliverables, rationale,
  and a live status table.
- `docs/architecture.md` — analyst/narrator design, hardware & network, model choices,
  data-source notes.
- `docs/inventory.md` — baseline of what's already metered (captured 2026-06-21).
- `docs/release-process.md` — versioning & GitHub release workflow.
- ADR-0001 — LLM narrates, Home Assistant analyses.
- ADR-0002 — repository as source of truth; YAML packages where version control matters.
- `docs/ai-context.md` — cold-start orientation map for AI sessions working on this repo.

### Phase 1 (Tier-1 consumption capture) — 2026-06-21
- `sensor.house_power_now` — `min_max(sum)` of harvi generation + grid CT (live whole-house W).
- `sensor.house_energy_total` — Riemann `integration` of `house_power_now` (whole-house kWh).
- `utility_meter` daily + monthly on house energy, washing machine, pool, and hot water (iBoost):
  `sensor.{house,washing_machine,pool,hot_water}_energy_{daily,monthly}`.
- Build log: `docs/phases/phase-1-consumption-capture.md`.

### Notes
- Pending in Phase 1: `statistics` baselines (after ~1–2 weeks of data) and Energy Dashboard wiring.

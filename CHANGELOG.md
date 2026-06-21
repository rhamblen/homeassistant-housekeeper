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

### Changed
- Energy Dashboard: grid **export** source switched from the myenergi daily-reset CT estimate
  to the metered Octopus export sensor (`…_current_total_export`), consistent with the import source.
- Docs: model choices reconciled to the **actually-installed** Ollama models (`llama3.1:8b` for
  narration/control, `minicpm-v` for vision); `qwen2.5:14b` reframed as an optional later upgrade.

### Added (Phase 2 — first narration working)
- Connected the **Ollama** integration (entry `01KVNASEQNK3PXX03ZE7TX943D`).
- Created **AI Task** agent `ai_task.ollama_ai_task_llama3_1` (model `llama3.1`, `num_ctx` 8192,
  `keep_alive` -1).
- `prompts/energy-snapshot.md` — first AI Task narration prompt, test-fired live and verified accurate.
- Build log: `docs/phases/phase-2-ai-narration.md`.
- Lesson captured: pre-compute facts in HA (e.g. import/export direction) — never let the 8B model
  interpret signs/arithmetic (ADR-0001).
- **Housekeeper dashboard** at `/ai-housekeeper`: live `house_power_now` gauge, per-device Today/
  Month tiles, 24h trend, and an **AI snapshot** card (`input_text.ai_energy_snapshot` +
  `script.refresh_energy_snapshot`) with a one-tap Refresh button.
- **Energy-flow layout**: sections for Energy in & out, Solar routing (generated → self-used /
  unused → returned / immersion), and Loads (incl. car charging). New derived template helpers
  `sensor.house_consumption_today`, `sensor.solar_unused_today`, `sensor.solar_self_used_today`
  (full-day, off the native myenergi `…_today` totals). Routing balances: generated = self-used +
  immersion + export.
- **Sankey flow diagram** on a new **Flow** view: Grid + Solar → House → Car / Immersion / Pool /
  Washing / Other-baseline (+ Solar → Grid export). Uses HACS `ha-sankey-chart` (browser
  hard-refresh needed) and new helper `sensor.other_baseline_today` (unmetered remainder). Kitchen/
  fridge/lighting fall into "Other / baseline" until metered.

### Notes
- Energy Dashboard audit: already well configured (solar/gas/grid + device breakdown). No battery.
- Pending in Phase 1: `statistics` baselines (after ~1–2 weeks of data); **FIT income** modelling
  (deferred — Richard is on the Feed-in Tariff; needs a dedicated generation + export income helper,
  not the dashboard export-price field).

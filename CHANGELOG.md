# Changelog

All notable changes to this project are documented here.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning: [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **Kitchen estimate v1** (`sensor.kitchen_estimated_power`) — first load-disaggregation slice:
  sum of assumed watts for Neff appliances that are "On". Carved out of `house_other_now`; added as
  a white/grey band on the 24h Consumption chart. Estimate only (tunable); accuracy path noted
  (oven operation-state/temperature) in the Phase 4 spec.

## [0.1.0] — 2026-06-22

First working release: a local-LLM "home manager" foundation on Home Assistant — consumption
capture, AI narration, and the Housekeeper dashboard (energy-flow Sankey, live gauges, history).

### Added — project & docs
- Repository scaffold: README, MIT license, changelog, `.gitignore`.
- `docs/project-plan.md` (8-phase roadmap + status table), `docs/architecture.md`,
  `docs/inventory.md`, `docs/release-process.md`, `docs/ai-context.md`.
- ADR-0001 (LLM narrates, HA analyses) and ADR-0002 (repo as source of truth).
- Phase build logs: `docs/phases/phase-1-…`, `phase-2-…`, and the `phase-4-load-disaggregation.md` spec.

### Added — Phase 1: consumption capture
- `sensor.house_power_now` (min_max sum of harvi generation + grid), `sensor.house_energy_total`
  (Riemann integration), and `utility_meter` daily+monthly for house / washing machine / pool / hot water.

### Added — Phase 2: AI narration
- **Ollama** integration + **AI Task** agent `ai_task.ollama_ai_task_llama3_1` (llama3.1, num_ctx 8192).
- `prompts/energy-snapshot.md`; `input_text.ai_energy_snapshot` + `script.refresh_energy_snapshot`.

### Added — Housekeeper dashboard (`/ai-housekeeper`)
- **Energy** view: live gauge, AI-snapshot card, energy in/out + solar-routing + loads tiles
  (helpers `house_consumption_today`, `solar_unused_today`, `solar_self_used_today`).
- **Flow** view: coloured Sankey (HACS `ha-sankey-chart`) — Grid + Solar → House / Immersion / Car
  / Grid export → loads; helpers `house_general_today`, `other_baseline_today`.
- **Live** view: real-time W gauges (car 7 kW, immersion/washing 4 kW) + Neff kitchen on/off tiles
  with icons; helper `house_other_now`.
- **History (24h)**: two ApexCharts **stacked-column** charts sharing one envelope — Supply
  (solar + grid) and Consumption-by-load (other/pool/washing/car/immersion); helper `solar_self_used_now`.

### Added — Phase 4 (down-payment, paused)
- Disaggregation spec; baseline learners `sensor.house_baseline_power` / `house_baseline_today`.

### Changed
- Energy Dashboard grid **export** source → metered Octopus export (consistent with import).
- Model choices reconciled to the **installed** Ollama models (`llama3.1`, `minicpm-v`).

### Notes
- **Immersion** is a real measurement via the **harvi CT3 clamp** through a template (not a dedicated
  iBoost integration, not the Zappi) — see `docs/inventory.md`.
- ApexCharts stacked **areas** need timestamp-aligned series; we used **columns** to sidestep that.
  Smooth areas later via one-clock "aligned" sampling helpers.
- Deferred / pending: `statistics` baselines settling; **FIT income** modelling; Phase 4 metering
  (kitchen/fridge/lighting still inside "Other"); curating exposed entities for conversational control.

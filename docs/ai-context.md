# ai-context

Orientation map for an AI (Claude) session working on this repo. **Not for end users.**
Dense and factual — read this first to work safely without re-deriving context.

## Repository purpose

Turn a large Home Assistant install into a proactive **home manager** driven by a local
LLM (Ollama). Core design: **HA is the analyst** (helpers/recorder do maths, anomaly
detection, history); **the LLM is the narrator** (explains/advises on a schedule or on
demand). Never ask the LLM to crunch raw numbers — see [ADR-0001](decisions/0001-llm-as-narrator-not-analyst.md).

## How to work in this repo (rules)

- **Read [project-plan.md](project-plan.md) first** — phases, status table, exit criteria.
- **Honour the ADRs** in [decisions/](decisions/). [ADR-0002](decisions/0002-repository-as-source-of-truth.md):
  repo = source of truth for intent + rationale; version-controlled config goes in
  `packages/` as YAML; **never hand-edit HA `.storage/`**.
- **Before creating any HA helper/automation/dashboard**, read the
  `home-assistant-best-practices` skill (prefer native helpers over templates; use
  `entity_id` not `device_id`; pick the right automation mode).
- **Create config via proper mechanisms** — MCP HA tools / UI / YAML packages — not by
  editing storage.
- **Update [CHANGELOG.md](../CHANGELOG.md) on every change** (`[Unreleased]`). Releases
  only when explicitly asked; follow [release-process.md](release-process.md).
- Add a build log under [phases/](phases/) as each phase is executed.

## Environment / key IDs

| Thing | Value |
|---|---|
| Home Assistant | `192.168.1.64`, HA 2026.7 (dev), 5,732 entities, 26 areas |
| Ollama | `http://192.168.1.32:11434` (Docker on Unraid UR1, br0) |
| GPUs | RTX 3090 24 GB (brain) + RTX 3060 12 GB (vision) on UR1 |
| Models (installed) | `llama3.1:8b` (tools — narration/control), `minicpm-v:7.6b` (vision). Also `mistral:7b`, `MrTails/Tails-assistant-ai-v3` (HA-tuned), `deepseek-r1:8b`. `qwen2.5:14b` = optional later upgrade. |
| Real-time power | myenergi harvi grid CT (NOT the smart meter) |
| Cost / tariff | Octopus (half-hourly, ~1 day delayed; no Home Mini) |
| Access to HA | via the `Home_Assistant` MCP tools in-session |

## Key entities (the ones phases build on)

| Entity | Role |
|---|---|
| `sensor.myenergi_harvi_11174172_power_ct_grid` | Live grid W (neg = export) |
| `sensor.myenergi_harvi_11174172_power_ct_generation` | Live solar W |
| `sensor.house_power_now` | **Built (Phase 1)** — min_max(sum) of generation + grid (W) |
| `sensor.house_energy_total` | **Built (Phase 1)** — Riemann integral of house_power_now (kWh) |
| `sensor.{house,washing_machine,pool,hot_water}_energy_{daily,monthly}` | **Built (Phase 1)** — utility_meter cycles |
| `sensor.pool_power_power` / `_energy` | Pool circuit (Shelly PM) |
| `sensor.washing_machine_energy` / `_machine_state` / `_job_state` / `_completion_time` | Samsung washer (SmartThings); energy reliable, live W coarse |
| `sensor.hot_water_diverted_energy_solar_iboost` | Immersion diversion (kWh) |
| `sensor.garden_water_meter_flow_rate` (L/h) + zone valves `switch.tap_*` | Leak watchdog (Phase 3) |
| `sensor.battery_critical_devices` / `sensor.battery_low_devices` | Existing battery roll-ups (analyst work already done) |

Full baseline in [inventory.md](inventory.md); design/hardware detail in [architecture.md](architecture.md).

## Build phases (mirror of the plan)

| Phase | Goal | Status |
|------:|------|--------|
| 0 | Foundations (Ollama connect, repo, exposed entities) | ☐ |
| 1 | Consumption capture (utility_meter / integration / statistics helpers, `house_power_now`) | ◐ Tier-1 built |
| 2 | AI narration (morning briefing, wash cost, energy review) | ☐ |
| 3 | Anomaly watchdogs (garden leak, device offline, left-on) | ☐ |
| 4 | Load fingerprinting / disaggregation | ☐ |
| 5 | Camera intelligence (vision) | ☐ |
| 6 | Conversational control + energy optimisation | ☐ |
| 7 | (Optional) per-circuit metering hardware | ☐ |

Keep this table and the one in [project-plan.md](project-plan.md) in sync.

## Gotchas

- Exposing all 5,732 entities to the LLM will break it — curate ~40–80 for Assist (Phase 0/6).
- Ollama container restart policy was `no`; both GPUs already carry baseline VRAM from other
  containers (usable headroom ~13 GB / ~6 GB, not the full 36 GB).
- The washer's instantaneous `power` (W) is unreliable — use the `energy` (kWh) delta for per-cycle cost.
- Garden water meter `faults` has been seen reading `empty_pipe_alarm,transducer_alarm` (likely no-flow/off-season).

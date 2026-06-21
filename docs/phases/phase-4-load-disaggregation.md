# Phase 4 — Load disaggregation (spec)

**Status:** ☐ Spec only. Down-payment helpers built 2026-06-21 (`house_baseline_power`,
`house_baseline_today`) — settling, not yet wired into the Sankey.

Goal: split the Sankey's big **"Other / baseline"** block into real loads, so the diagram shows
where the unmonitored consumption actually goes (kitchen, fridge/freezer, dryer, etc.).

## Architecture — it's an algorithm, not an LLM guess
Richard's framing: *"identify what's turned on when the power increases, then calculate the
consumption."* That is **NILM (non-intrusive load monitoring)** — a **deterministic** detection +
attribution problem. Per [ADR-0001](../decisions/0001-llm-as-narrator-not-analyst.md), the engine is
HA logic (and possibly n8n for the stateful loop); **the LLM only narrates the result** ("the dryer
likely ran 18:40–19:30, ~2.3 kWh"). We do **not** ask the model to guess which appliance turned on.

## Attribution waterfall (richard's "some track power, many don't")
For any rise in `sensor.house_power_now`, attribute in priority order:

1. **Self-metered (exact)** — subtract these from the total, no inference needed:
   washing machine (SmartThings energy), pool (Shelly PM), car (Zappi), immersion (iBoost).
2. **State-signalled, energy estimated** — Neff via Home Connect report **on/off, not watts**:
   ovens L/R, hob, extractor, warming drawers, dishwasher. We know **when** they switch on, so
   attribute the coincident house-power step to them; energy = attributed ΔP × on-duration.
   - Dishwasher: its own cycle (late evening, < 2h). Group with or separate from baseline — NOT baseline.
3. **Baseline (real)** — fridge/freezer + standby = the **overnight minimum** of house load
   (`sensor.house_baseline_power`, value_min over 24h). Always-on floor.
4. **Inferred / unidentified residual** = total − (1) − (2) − (3). High-drain steps with **no**
   state signal land here.
   - **Tumble dryer (no signal):** dumb appliance, high-drain. Heuristic — a sustained > ~2 kW step
     with no matching state change, *especially within ~2h after the washing machine finishes*
     (same-day, usually within an hour, not guaranteed) → label "likely tumble dryer";
     energy = step ΔP × duration.

## Engine mechanics
- **Step detector** on `house_power_now` (a `derivative`/`threshold` helper, or trend) fires on
  significant ± changes.
- On a **+step**: snapshot which known devices changed state in the same few seconds (Home Connect
  on-transition) and which self-metered loads moved → attribute the ΔP.
- Track each attributed load's start time + power; on its **−step** (off), energy = power × duration; log it.
- Unattributed +steps → unidentified bucket / dryer heuristic.
- Likely home for the stateful loop: **n8n** (flagged earlier as where it earns its place) with HA
  feeding events; or HA automations for the simpler cases. LLM narrates.

## Honest accuracy
Estimates, not measurement. Oven draw varies hugely (preheat ~2 kW vs maintain ~0.5 kW);
simultaneous changes confound attribution; the dryer is pure inference. The **accurate** route is
per-circuit metering (Shelly EM / Pro 3EM, or Emporia Vue) on the kitchen ring + utility circuits —
recommended for the loads that matter. Disaggregation is the "good enough, no hardware" path.

## Inputs available
- `sensor.house_power_now`, `sensor.house_baseline_power`.
- Self-metered: `sensor.washing_machine_energy`, `sensor.pool_power_energy`,
  `sensor.myenergi_zappi_20894508_energy_used_today`, `sensor.hot_water_diverted_energy_solar_iboost`.
- Washing machine cycle: `sensor.washing_machine_job_state` / `_machine_state` / `_completion_time`.
- Neff Home Connect power-state sensors (ovens, hob, extractor, warming drawers, dishwasher, freezer).

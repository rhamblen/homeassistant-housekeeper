# Phase 4 — Load disaggregation (spec)

**Status:** ◐ Spec + first slice. Down-payment helpers built 2026-06-21 (`house_baseline_power`,
`house_baseline_today`). **Kitchen estimate v1 built 2026-06-22** — see below.

## Kitchen estimate v1 (2026-06-22)
`sensor.kitchen_estimated_power` (W) = sum of **assumed watts for each Neff appliance whose
power-state == `…PowerState.On`**: Oven L/R 1500 each, Hob 1500, Dishwasher 1000, Extractor 120,
Warming tray L/R 300 each (all tunable). Carved out of `house_other_now` (which now also subtracts
kitchen) so the consumption stack still sums to the house total — added as a white/grey band on the
24h Consumption chart (order: Other, Pool, **Kitchen**, Washing, Car, Immersion).
- **It's an estimate, not measured** — state × fixed watts; ignores oven thermostat duty-cycle.
- No back-history (new sensor) — the band fills forward.

### Kitchen accuracy v2 — learn real watts from the power step (confirmed approach)
Replace the fixed assumptions with **measured** draw: when a Neff appliance switches to `…PowerState.On`,
snapshot `house_power_now` **just before** the transition and again **a few seconds after** it settles;
the **delta (after − before) = that appliance's real power**. The matching **drop at power-off** confirms it.
Bank that delta each time and **average over many switch-ons** to build a learned per-appliance wattage
that supersedes the estimate.
- **Only clean/isolated steps count** — discard any sample where another tracked load (EV, immersion,
  pool, etc.) changed within the same few seconds, else the delta is contaminated. Hence learn-over-time.
- **Ovens modulate** — the on-delta is the *element* peak (~2 kW); combine with cavity-temperature /
  operation-state (both exposed) to model heating vs coasting for energy (not just peak power).
- Deterministic capture loop (HA/n8n); LLM narrates only (ADR-0001).

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

## Extending to rooms / lights / device level
The same edge-correlation generalises: every `light`/`switch` toggle is a power edge. On an
**isolated** state change (no other tracked entity changed within a few seconds), capture
ΔP and fold it into a **running average per entity** → a learned-watts **library**. Because
entities carry an `area_id`, summing learned device watts by area yields **room-level**
consumption. "Learning" = a deterministic capture loop (HA/n8n) maintaining the averages;
the LLM narrates and helps disambiguate — it does not do the attribution.

### Honest feasibility — the signal-to-noise wall
Whole-house CT NILM works **well for big/medium distinct loads** (kettle, oven, dryer, pump,
EV) and **poorly for small ones**. A 7 W LED toggling is essentially invisible against a house
that swings kilowatts (fridge/freezer/pool/EV cycling) — its edge is below the noise floor, so
passive learning of small lights is unreliable. Where small-device learning *does* work:
- the change happens **in isolation at a quiet time**, or
- the device (smart plug / smart switch) **reports its own power** — then it's exact, tier-1.

So the realistic trajectory: learn the **big/medium** loads reliably from whole-house power;
get **small/room** detail only from devices that self-meter or from per-circuit CT. Manage
expectations — "every light" is not reliably learnable from one CT.

### Pragmatic build order
1. Capture loop records **isolated** edges → per-entity running-average library (starts broad, refines over weeks).
2. Use any **self-reported power** (smart plugs, Shelly, etc.) directly — exact nodes.
3. Roll learned device watts up by `area_id` → room estimates (clearly flagged estimated).
4. Per-circuit CT (Shelly Pro 3EM / Emporia) for the rooms/circuits that matter → turns estimates into measurements.

## Inputs available
- `sensor.house_power_now`, `sensor.house_baseline_power`.
- Self-metered: `sensor.washing_machine_energy`, `sensor.pool_power_energy`,
  `sensor.myenergi_zappi_20894508_energy_used_today`, `sensor.hot_water_diverted_energy_solar_iboost`.
- Washing machine cycle: `sensor.washing_machine_job_state` / `_machine_state` / `_completion_time`.
- Neff Home Connect power-state sensors (ovens, hob, extractor, warming drawers, dishwasher, freezer).

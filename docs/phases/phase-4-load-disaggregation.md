# Phase 4 — Load disaggregation (spec)

**Status:** ◐ In progress. Down-payment helpers built 2026-06-21 (`house_baseline_power`,
`house_baseline_today`). Kitchen estimate v1 built 2026-06-22. **Kitchen v2 (per-session
measured draw) built 2026-06-24** — see below.

## Kitchen estimate v1 (2026-06-22)
`sensor.kitchen_estimated_power` (W) = sum of **assumed watts for each Neff appliance whose
power-state == `…PowerState.On`**: Oven L/R 1500 each, Hob 1500, Dishwasher 1000, Extractor 120,
Warming tray L/R 300 each (all tunable). Carved out of `house_other_now` (which now also subtracts
kitchen) so the consumption stack still sums to the house total — added as a white/grey band on the
24h Consumption chart (order: Other, Pool, **Kitchen**, Washing, Car, Immersion).
- **It's an estimate, not measured** — state × fixed watts; ignores oven thermostat duty-cycle.
- No back-history (new sensor) — the band fills forward.

## Kitchen estimate v2 — per-session measured draw (2026-06-24)

**Built and live.** Replaces fixed assumed watts with a measured delta captured at each switch-on.

### Design rationale
Appliance draw varies by usage — an oven preheating draws ~2 kW, maintaining ~0.5 kW. A
long-term average of either is wrong for the current session. The correct model: measure the
actual house-load step at the moment this appliance turns on, and use *that* value for the
duration of this session. Formula:

```
Hr     = house_power_now − car − pool − immersion   (house running load)
Hr_net = Hr − washing_machine                       (washing is sub-metered → known, not a contaminant)
O      = Hr_net  when kitchen is idle               (other background)
K      = Hr_net − O  when any appliance is on       (kitchen contribution this session)
```

Per-device: at each `…PowerState.On` trigger, delta = `Hr_net_after − Hr_net_before` (8 s
settle). That delta is this appliance's draw for *this cycle*. Next switch-on overwrites it.

### HA objects
- **7 `input_number.kitchen_learned_watts_{oven_l,oven_r,hob,dishwasher,extractor,warming_l,warming_r}`**
  — store the most-recent switch-on delta per appliance. Seeded at v1 assumed values
  (1500/1500/1500/1000/120/300/300 W) as cold-start fallback until first real measurement.
- **`sensor.kitchen_energy_total`** (integration, kWh, left method) — Riemann integral of
  `sensor.kitchen_estimated_power`. Feeds the utility_meters below.
- **`sensor.kitchen_energy_daily`** / **`sensor.kitchen_energy_monthly`** (utility_meters) — daily
  and monthly kWh totals for kitchen. Subtracted from `sensor.other_baseline_today` /
  `sensor.other_baseline_monthly` so "Other" always excludes kitchen.
- **Automation `automation.kitchen_measure_appliance_draw_from_power_step`** — `mode: parallel`,
  7 state → `…PowerState.On` triggers. At trigger: capture `Hr_net_before`. After 8 s: capture
  `Hr_net_after`, compute `delta`. Contamination guard: discard if EV/pool/immersion moved > 200 W
  (washing is subtracted, so its changes don't contaminate). If `30 < delta < 5000 W`: write
  `delta` directly to the appliance's `kitchen_learned_watts_*` helper (simple assignment, no average).
- **`sensor.kitchen_estimated_power`** — unchanged in structure; reads `kitchen_learned_watts_*`,
  each gated by `is_state(neff_entity, '…PowerState.On')`.

### Behaviour
- **First run:** uses v1 seed values.
- **After first switch-on:** uses the delta measured at that switch-on for the rest of the session.
- **Next session:** overwrites with a fresh delta — always reflects the current usage, not a historical mean.
- The stored value persists across HA restarts and serves as the opening estimate until the next
  clean measurement.

### Kitchen accuracy v2 — per-session delta (built, see section above)
Built 2026-06-24. See the v2 section above for the full design. Key principle: each switch-on
captures a fresh delta (`Hr_net_after − Hr_net_before`); that session's actual draw is used for
the duration, not averaged with prior sessions. Washing machine is subtracted live (not guarded).
EV/pool/immersion changes > 200 W discard the sample. Next step for higher accuracy: oven
operation-state / cavity temperature to model duty-cycle within a session.

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

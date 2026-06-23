# Inventory — what's already measured

Baseline snapshot of the data we have to build on. Captured 2026-06-21 from the
live system. Update as integrations change.

## System scale
- HA 2026.7 (dev), **5,732 entities**, 39 domains, **26 areas**, ~35 automations.

## Already metered (energy/power) — Phase 1 targets

| Device / circuit | Source | Key entities | Notes |
|---|---|---|---|
| Whole-house grid | myenergi harvi | `sensor.myenergi_harvi_11174172_power_ct_grid` (W, live), `..._power_ct_generation`, `..._power_ct_storage` | Real-time; neg grid = export |
| myenergi hub | myenergi | `sensor.myenergi_hub_20111566_power_import` / `_export` / `_generation` | Live |
| Washing machine | SmartThings (Samsung) | `sensor.washing_machine_energy` (kWh), `_machine_state`, `_job_state`, `_completion_time` | Energy reliable; live W coarse |
| Pool circuit | Shelly PM | `sensor.pool_power_power` (W), `sensor.pool_power_energy` (kWh) | **SEPARATE supply from the house meter** — the harvi CT clamp (and Octopus) do NOT measure pool. Shelly measures the pool sub-board: heat pump + pump + lights. |
| Pool heat pump | template (derived from Shelly) | `sensor.pool_heat_pump_power` (W) = `max(0, pool_power_power − 252 − (26 if lights on))` | Subtracts pool pump (252 W fixed) and lights (26 W when on). Confirmed topology 2026-06-23. |
| EV charger | myenergi zappi | `sensor.myenergi_zappi_20894508_power_ct_internal_load` | Charge mode Eco+ |
| EV (car) | Audi connect | `sensor.audi_q4_e_tron_charging_power` | |
| Hot water diversion (immersion) | **harvi CT3 clamp** via template (NOT a real iBoost integration, NOT the Zappi) | `sensor.hot_water_diverted_solar_iboost` (W) = `-1 × ..._power_ct_storage`; `sensor.hot_water_diverted_energy_solar_iboost` (kWh) | CT3 ("storage", no battery) is clamped on the immersion diverter. Real CT measurement, just surfaced via template. Sub-meter of a load already in `house_power_now` (gen+grid) — do NOT add CT3 as a source. Verify clamp by running immersion → should jump to ~3 kW. |

## Water
- **Garden water meter**: `sensor.garden_water_meter_flow_rate` (L/h), `_daily_consumption`,
  `_month_consumption`, `_water_consumed`, `_faults`. (Faults seen reading
  `empty_pipe_alarm,transducer_alarm` — likely no-flow/off-season; verify.)
- **Garden zone valves** (Zigbee, the real ones):
  `switch.tap_lhs_upper_lawn_blue`, `switch.tap_lhs_lower_lawn_green`,
  `switch.tap_lhs_fence_green_yellow_2`, `switch.tap_rhs_fence_2`, `switch.tap_veg_trug`.
  (Ignore `..._auto_close_when_water_shortage` config switches and stale `unavailable` dupes.)
- B-hyve irrigation system also present.

## Existing derived/aggregate sensors (analyst work already done)
- `sensor.battery_critical_devices`, `sensor.battery_low_devices` — battery roll-ups.
- myenergi integration/template helpers: "Solar Generation Energy", "Storage CT3
  Accumulated Energy", "Hot Water Diverted (Solar iBoost)".

## Cameras (Phase 5)
- Reolink with object detection: Drive, Front Door, Side Entrance, Pool, Lawn, Patio,
  Bird cam, Garden.

## Appliances reporting state (Phase 4 passive fingerprinting)
- Neff via Home Connect: ovens, hob, hood, freezer, warming drawers, dishwasher
  (report power *state* on/standby/off, not watts).

## Energy / tariff
- Octopus Energy: electricity + gas meters, rates, Saving Sessions / Greener Nights
  calendars, Intelligent dispatching (Zappi). Half-hourly, ~1 day delayed.

## Delivery surfaces
- Mobile `notify`: Richard's iPhone(s), iPads. Sonos (Beam, etc.) + Alexa dots for TTS.

## Pool heat pump COP — blocked, open issue

**Method (agreed 2026-06-23):** energy-balance over a heating session —
`COP = (pool_volume_L × 4.186 × pool_temp_rise_°C) ÷ heat_pump_kWh_session`.
No outlet temp or flow meter needed. Needs only pool volume (unknown) + pool temp
(`sensor.pool_chem_temperature` / heat pump inlet, 1 °C resolution).

**Blocked because:**
- The Tuya heat pump exposes inlet water temp, compressor current, coil/pipe/ambient/discharge
  temps — but **no outlet water temp** and **no flow meter**.
- 1 °C resolution on inlet temp means the delta over a short session is too coarse for a
  reliable single-session COP; a multi-hour session raising the pool 3+ °C gives a usable figure.
- Pool volume not yet confirmed.

**To unblock (either option):**
1. *Hardware (accurate):* outlet-temp probe + flow meter on the return pipe → instantaneous COP.
2. *Software (approximate):* capture pool inlet temp at heat-pump start + stop; use pool volume
   × 4.186 × ΔT ÷ kWh session; average over several sessions. Need pool volume from owner.
   Then add `utility_meter` on `sensor.pool_heat_pump_power` for per-session kWh.

## Not measured (gaps)
- Per-room / per-circuit electricity (no CT clamps beyond harvi at the board) → Phase 7
  or inference (Phase 4).
- Humidity per room — to confirm.
- **Pool COP** — see open issue above.

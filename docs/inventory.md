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
| Pool circuit | Shelly PM | `sensor.pool_power_power` (W), `sensor.pool_power_energy` (kWh) | Live |
| Pool heat pump | integration | `sensor.pool_heat_pump_power` (W) | Live |
| EV charger | myenergi zappi | `sensor.myenergi_zappi_20894508_power_ct_internal_load` | Charge mode Eco+ |
| EV (car) | Audi connect | `sensor.audi_q4_e_tron_charging_power` | |
| Hot water diversion | iBoost / iBoost solar | `sensor.hot_water_diverted_energy_solar_iboost` (kWh) | Solar surplus → immersion |

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

## Not measured (gaps)
- Per-room / per-circuit electricity (no CT clamps beyond harvi at the board) → Phase 7
  or inference (Phase 4).
- Humidity per room — to confirm.

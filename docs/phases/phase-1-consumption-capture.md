# Phase 1 — Consumption capture (build log)

**Status:** ◐ In progress — Tier-1 helpers built 2026-06-21. Statistics baselines and Energy
Dashboard wiring still pending.

## What we built

All created via the HA config-flow (MCP `ha_config_set_helper`), per
[ADR-0002](../decisions/0002-repository-as-source-of-truth.md) — no `.storage` hand-edits,
no YAML `template:`.

### Live whole-house power
| Entity | Type | Definition |
|---|---|---|
| `sensor.house_power_now` | `min_max` (type **sum**) | `generation + grid` CT from the harvi (W) |

Sources: `sensor.myenergi_harvi_11174172_power_ct_generation` +
`sensor.myenergi_harvi_11174172_power_ct_grid` (grid negative = export, so the sum is true
household load). Chose `min_max(sum)` over a template sensor per the best-practices helper guide.

### Whole-house energy
| Entity | Type | Definition |
|---|---|---|
| `sensor.house_energy_total` | `integration` (Riemann) | `house_power_now` → kWh; method `left`, `unit_prefix=k`, `unit_time=h`, `max_sub_interval=1min` |

Derived because there is **no cumulative whole-house import-kWh sensor** (Octopus is half-hourly /
delayed). Riemann integral of the live load gives a real-time running total.

### Utility meters (daily + monthly)
| Source | Daily entity | Monthly entity |
|---|---|---|
| `sensor.house_energy_total` | `sensor.house_energy_daily` | `sensor.house_energy_monthly` |
| `sensor.washing_machine_energy` | `sensor.washing_machine_energy_daily` | `sensor.washing_machine_energy_monthly` |
| `sensor.pool_power_energy` | `sensor.pool_energy_daily` | `sensor.pool_energy_monthly` |
| `sensor.hot_water_diverted_energy_solar_iboost` | `sensor.hot_water_energy_daily` | `sensor.hot_water_energy_monthly` |

## How to reproduce
1. `min_max` helper, type `sum`, entity_ids = the two harvi CT sensors → House Power Now.
2. `integration` helper on `sensor.house_power_now` (left / k / h / max_sub_interval 1 min) → House Energy Total.
3. `utility_meter` helper per source × {daily, monthly}.

## Decisions & surprises
- **Storage CT excluded** from `house_power_now` (≈ −13 W, no clear battery). Revisit if a battery is added.
- **Pool/washer meter entity_ids renamed** — HA auto-generated `sensor.pool_pool_power_pool_energy_*`
  and `sensor.laundry_room_washing_machine_energy_*`; renamed to clean slugs while they had zero
  consumers (safe per safe-refactoring).
- **Fresh utility meters read `unknown`** until their source next reports — expected, not a fault.
- **Watch:** washer & pool daily meters reported first `next_reset` on the 23rd (not the 22nd),
  probably from the mid-cycle rename. Confirm they reset daily after the first cycle.

## Verification
- `house_power_now` = 1600 W, matched live `generation + grid`.
- `house_energy_total` accruing (0.05 kWh within ~1 min of creation).
- `house_energy_daily` collecting, `next_reset` = next local midnight.

## Still pending in Phase 1
- `statistics` baseline helpers (mean/stdev) — defer until meters accumulate ~1–2 weeks of history.
- Wire `house_energy_total` + device meters into the **Energy Dashboard** (device consumption).
- Optional: per-device `integration` helpers for any other power-only sensors.

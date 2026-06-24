# Changelog

All notable changes to this project are documented here.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning: [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **Irrigation plan (daily-summary section)** — `prompts/irrigation-plan.md`: an HA-rendered fact
  block for the morning briefing listing what's due to be watered over the **next 24 h** (zones,
  start times, durations across the two garden schedules), flagging **today's** runs that will be
  **skipped because of rain**, reporting an earlier-today run that **was stopped because of rain**,
  and giving **tomorrow's** runs a forecast-based verdict (Met Office daily precip probability vs
  `garden_rain_threshold`) since tomorrow's cancel isn't decided yet. Pure narration over existing
  helpers (the `garden2_{a,b}_*` schedule helpers, the `switch.tap_*` zone valves, the
  `garden_rain_cancel` / `garden_winter_shutdown` gates, `weather.met_office_fleet`) — no new HA
  objects, no schedule logic in the LLM (ADR-0001). Verified live + scenarios via `ha_eval_template`
  (2026-06-23).
- **Irrigation card live on the Housekeeper Prompts tab** — the irrigation plan is now shown as a
  deterministic `markdown` card (rendered by HA, not narrated by the LLM — it's structured data, and
  this guarantees zone names are never dropped). New supporting objects:
  `input_number.garden_rain_prob_tomorrow` + `automation.garden_forecast_rain_recorder` (every 30 min
  / on start: `weather.get_forecasts` daily → tomorrow's precip probability), letting the card show a
  rain verdict for tomorrow's runs without an LLM round-trip. Added as the third section on the
  **Prompts** tab with a **Refresh forecast** button.
- **Evening charge advisory** — first proactive AI brief. Nightly *"should I charge the Audi
  tonight?"* suggestion: HA gathers the facts (Audi SoC / plug / location, tomorrow's solar
  forecast, tomorrow's diary from `calendar.richard` + `calendar.ruth`, Octopus unit rate), the
  LLM phrases a 1–2 sentence suggestion — **advisory only**, no charging action (ADR-0001).
  Pieces: `prompts/charge-advisory.md`; `input_text.ai_charge_advisory`;
  `script.charge_advisory_generate` (button-testable); `automation.evening_charge_advisory`
  (21:00 → script → **Slack `#home-assistant`** via `notify.basingbourne` + mobile push
  `notify.mobile_app_rich_iphone_15`); a **"Charge tonight?"** section on `/ai-housekeeper`.
  Verified end-to-end 2026-06-22 — the existing Slack integration was reused, no token setup needed.
- **24h history refinements** — added the Kitchen estimate band; moved Immersion to the base so its
  negative noise renders below zero; added **Grid export** as a positive consumption band so both
  charts balance to one envelope (`supply = generation + import` = `consumption = loads + export`).
- **Kitchen estimate v1** (`sensor.kitchen_estimated_power`) — first load-disaggregation slice:
  sum of assumed watts for Neff appliances that are "On". Carved out of `house_other_now`; added as
  a white/grey band on the 24h Consumption chart. Estimate only (tunable); accuracy path noted
  (oven operation-state/temperature) in the Phase 4 spec.

- **Kitchen energy metering (daily + monthly)** — `sensor.kitchen_energy_total` (Riemann integral,
  kWh, left method) + `sensor.kitchen_energy_daily` / `sensor.kitchen_energy_monthly` (utility_meters).
  `sensor.other_baseline_today` and `sensor.other_baseline_monthly` updated to subtract kitchen
  so house "Other" shrinks accordingly and energy balances are preserved. Dashboard: **Loads (today)**
  and **This month** sections each gain a "Kitchen (Neff)" tile; Sankey gains a Kitchen node carved
  out of the "Other" leaf; "House (kitchen + other)" tile labels simplified to "Other".
- **Kitchen estimate v2** — `sensor.kitchen_estimated_power` now uses **per-session measured draw**
  instead of fixed assumed watts. Formula: `Hr = house_power_now − car − pool − immersion`;
  `Hr_net = Hr − washing` (SmartThings sub-metered, so already known); when all kitchen appliances
  are off `Hr_net = O` (other baseline); when on `K = Hr_net − O`. In practice: automation
  `automation.kitchen_measure_appliance_draw_from_power_step` (mode: parallel) fires on each Neff
  appliance → `…PowerState.On`, snapshots `Hr_net_before`, waits 8 s, computes
  `delta = Hr_net_after − Hr_net_before` — that delta is the appliance's **actual draw this session**
  and is written directly to `input_number.kitchen_learned_watts_{appliance}`. Washing machine
  is subtracted in both readings (not a contaminant); EV/pool/immersion moves > 200 W still
  discard the sample. The stored value persists and is used as the opening estimate on the next
  switch-on until overwritten. 7 `input_number` helpers (seeded at v1 assumed values as cold-start
  fallback). Estimate is always from this cycle's real measurement, not a long-term average.

### Notes / open issues
- **Pool sub-metering (2026-06-23)** — pool circuit is on the **house supply** (harvi sees it);
  the Shelly PM sub-meters the pool circuit specifically (heat pump + pump + lights), the same
  way SmartThings sub-meters the washing machine. `pool_power_power` is correctly subtracted in
  `house_other_now` and `other_baseline_today`. Pool is a child of House in the Sankey.
  `sensor.pool_heat_pump_power` = Shelly − 252 W pump − 26 W lights.
- **Pool heat pump COP — blocked** — Tuya exposes inlet water temp, compressor current, coil
  temps but no outlet temp and no flow meter. COP method agreed: pool volume × 4.186 × ΔT_pool
  ÷ heat-pump kWh over a session. Unblock paths: (a) outlet probe + flow meter hardware;
  (b) use inlet temp rise (whole-session energy balance, needs pool volume). See `docs/inventory.md`.

### Changed
- **Housekeeper dashboard reorganised** — added a dedicated **Prompts** tab as the first view,
  gathering all AI narration cards (AI snapshot + Charge tonight? + Irrigation); the **Energy** tab
  now carries no prompt cards. New tab order: **Prompts → Energy → Live → Flow** (Sankey).
- **Charge advisory projects battery-on-arrival-home when away** — instead of an away/home guard, the
  script computes km-per-% (`range / SoC`) and subtracts the drive home (crow-flies `distance()` ×
  **1.5** road factor) to estimate the SoC when the car gets home, then decides off that band. Removes
  the "say nothing if away" behaviour. Verified live with the car ~15 km out (SoC 79% → ~74%).
- **Single "Refresh all advisories" button** — the per-card refresh buttons (snapshot / advisory /
  forecast) are replaced by one button at the top of the Prompts tab, running new
  `script.refresh_all_advisories` (energy snapshot + charge advisory + garden rain forecast).
- **Irrigation card now reports "Since midnight" too** — alongside *Next 24 hours*, it shows whether
  each past-due scheduled run **watered as scheduled** or the day **was cancelled because of rain**
  (schedule-inferred; the Tuya `…_daily_irrigation_volume` sensors are stale/non-resetting so are not
  used). Heading simplified to **Irrigation**.

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

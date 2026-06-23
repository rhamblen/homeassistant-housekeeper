# Prompt — Irrigation Plan (Phase 2, daily-summary section)

An **Irrigation** section for the daily AI summary, in two parts:
- **Since midnight** — what already happened today: each scheduled run **watered as scheduled**, or
  the day's watering **was cancelled because of rain**.
- **Next 24 hours** — what is due to be watered, at what time and for how long, whether today's runs
  will be **skipped because of rain**, and a forecast-based verdict for tomorrow's runs.

Same shape as the sibling briefs ([energy-snapshot.md](energy-snapshot.md),
[charge-advisory.md](charge-advisory.md)): **HA computes every fact**, the LLM only phrases it
(ADR-0001). Nothing here asks the 8B model to read a schedule or reason about rain — the fact block
below resolves all of that in HA and hands the model finished lines.

## Live implementation (built 2026-06-23) — deterministic card, not LLM

On reflection this is **structured data, not prose**, so per ADR-0001 it is rendered by HA, not
narrated by the 8B model (which also guarantees zone names are never paraphrased away — a concern
raised in build). It is live on the **Prompts** tab of `/ai-housekeeper` as a `markdown` card whose
`content` is the fact block below. Because a card template cannot call `weather.get_forecasts`,
tomorrow's rain figure is fed in via a tiny helper + recorder:

- `input_number.garden_rain_prob_tomorrow` (%) — tomorrow's daily precip probability.
- `automation.garden_forecast_rain_recorder` — every 30 min + on HA start: `weather.get_forecasts`
  (daily) → writes tomorrow's `precipitation_probability` to the helper. Mirrors the existing
  `garden_rain_recorder` pattern.
- Prompts-tab card: heading **Irrigation** + the markdown fact block. The card re-renders live (uses
  `now()`), so no output helper/`input_text` is needed and there is no 255-char limit. It has **no
  own refresh button** — the shared **Refresh all advisories** button at the top of the Prompts tab
  (`script.refresh_all_advisories`) re-pulls the forecast (among the other briefs).

The live card renders **two sections**: **Since midnight** (past runs — *watered as scheduled* if
rain-cancel was off, else *cancelled because of rain*) and **Next 24 hours** (upcoming runs with the
rain verdict). "Watered" is inferred from the schedule (a past-due armed run on an enabled day), not
from valve telemetry — the Tuya `…_daily_irrigation_volume` sensors proved stale/non-resetting
(unchanged for days, periodically `unavailable`), so they are deliberately not used.

The card's fact block reads `rn.tom` from `input_number.garden_rain_prob_tomorrow` (not a script-local
`fc`) and uses markdown **bold**. The block below matches the live card.

## How it's wired (alternatives)

Designed to drop into the **morning briefing** as one section (energy-snapshot.md flagged
*"water"* as a planned morning-briefing topic). Two ways to use it:

- **As a section** — paste the *fact block* into the morning-briefing generate script's `instructions`
  under an `Irrigation:` heading, alongside the other fact blocks. The briefing prompt then narrates
  all sections together.
- **Standalone** (optional, if irrigation ever wants its own push) — follow the charge-advisory
  pattern: `input_text.ai_irrigation_plan` (output mirror) + `script.irrigation_plan_generate`
  (builds the fact block, calls `ai_task.generate_data`, writes the helper) +
  `automation.morning_irrigation_brief` (a `time` trigger → script → notify).

**Prerequisite step (tomorrow's rain outlook).** Today's runs use the live `garden_rain_cancel`
flag (the real decision, made ~30 min before each run). Tomorrow's cancel hasn't been computed yet,
so the block instead uses the **forecast** — the same source `garden_rain_auto_cancel_check` acts
on. The generate script must therefore fetch the daily forecast into `fc` *before* rendering the
fact block (one extra action, identical to the auto-cancel automation):

```yaml
- action: weather.get_forecasts
  target: { entity_id: weather.met_office_fleet }
  data: { type: daily }
  response_variable: fc
```

> **Jinja caveat (Phase 2 lesson):** in an automation/script the templates below render against live
> state automatically. In a direct service/API call they are NOT rendered — pre-render (e.g.
> `ha_eval_template`) and pass the finished string. The fact block expects `fc` (above) to be in
> scope; if it is missing, tomorrow's line falls back to "(rain forecast unavailable)".

## The garden system (what the facts come from)

- **5 zones**, each a Tuya tap valve: `switch.tap_lhs_upper_lawn_blue`,
  `switch.tap_lhs_lower_lawn_green`, `switch.tap_lhs_fence_green_yellow_2`, `switch.tap_rhs_fence_2`,
  `switch.tap_veg_trug`. Zone names are read from each switch's friendly name (strip the `Tap - `
  prefix) — no hard-coded map.
- **2 schedules, A & B** (`automation.garden_watering_schedule_{a,b}_mk2` → `script.garden2_{a,b}_watering_sequence`).
  Each schedule is described entirely by helpers: an armed flag, a start time, per-day enable
  booleans, and up to 5 valve slots (tap entity + duration in minutes; duration 0 = slot unused).
- **Rain / season gate** (`automation.garden_rain_auto_cancel_check`): 30 min before either start it
  sets `input_boolean.garden_rain_cancel` **on** if it rained in the last 12 h **or** forecast rain
  probability ≥ `input_number.garden_rain_threshold` (next 24 h, `weather.met_office_fleet`). The flag
  is cleared overnight by `automation.garden_rain_cancel_daily_reset`, so it only ever suppresses
  **today's** runs — tomorrow's are listed as planned. `input_boolean.garden_winter_shutdown` is a
  hard off-switch for the whole system.

## Entities used

| Fact | Entity |
|---|---|
| Schedule armed | `input_boolean.garden2_{a,b}_schedule_armed` |
| Start time | `input_select.garden2_{a,b}_water_start_time` |
| Day enabled | `input_boolean.garden2_{a,b}_water_{mon..sun}` |
| Valve slot → zone | `input_text.garden2_{a,b}_valve_{1..5}_entity` (a `switch.tap_*`) |
| Valve slot → minutes | `input_number.garden2_{a,b}_valve_{1..5}_duration` |
| Rain cancel (today) | `input_boolean.garden_rain_cancel` |
| Rain threshold (%) | `input_number.garden_rain_threshold` |
| Last rain recorded | `input_datetime.garden_last_rain` |
| Winter shutdown | `input_boolean.garden_winter_shutdown` |
| Tomorrow's rain forecast | `weather.met_office_fleet` via `weather.get_forecasts` (`type: daily`) → `fc` |

## Fact block (HA-rendered — paste into the briefing's `instructions`)

This is the **live card** content (reads tomorrow's % from `input_number.garden_rain_prob_tomorrow`).
Two sections: **Since midnight** (past-due runs — *watered as scheduled*, or *cancelled because of
rain* if the day's rain-cancel is on) and **Next 24 hours** (today's still-upcoming runs + tomorrow's
runs within 24 h, with the rain verdict). Winter shutdown short-circuits both.

> For the **morning-briefing / standalone** wirings (no helper), swap the `rn` block for the
> `weather.get_forecasts` → `fc` extraction shown under *Prerequisite step* above.

```jinja
{%- set wd = ['mon','tue','wed','thu','fri','sat','sun'] -%}
{%- set winter = is_state('input_boolean.garden_winter_shutdown','on') -%}
{%- set rain_off = is_state('input_boolean.garden_rain_cancel','on') -%}
{%- set threshold = states('input_number.garden_rain_threshold')|int(40) -%}
{%- set rn = namespace(tom = states('input_number.garden_rain_prob_tomorrow')|int(-1)) -%}
{%- set ns = namespace(upcoming=[], past=[]) -%}
{%- for s in ['a','b'] -%}
  {%- set armed = is_state('input_boolean.garden2_' ~ s ~ '_schedule_armed','on') -%}
  {%- set start = states('input_select.garden2_' ~ s ~ '_water_start_time') -%}
  {%- if armed and start not in ['unknown','unavailable',''] -%}
    {%- set zns = namespace(z=[], mins=0) -%}
    {%- for n in [1,2,3,4,5] -%}
      {%- set e = states('input_text.garden2_' ~ s ~ '_valve_' ~ n ~ '_entity') -%}
      {%- set d = states('input_number.garden2_' ~ s ~ '_valve_' ~ n ~ '_duration')|int(0) -%}
      {%- if d > 0 and e not in ['unknown','unavailable',''] -%}
        {%- set zname = (state_attr(e,'friendly_name') or e) | replace('Tap - ','') -%}
        {%- set zns.z = zns.z + [zname ~ ' (' ~ d ~ ' min)'] -%}
        {%- set zns.mins = zns.mins + d -%}
      {%- endif -%}
    {%- endfor -%}
    {%- if zns.z | count > 0 -%}
      {%- if is_state('input_boolean.garden2_' ~ s ~ '_water_' ~ wd[now().weekday()], 'on') -%}
        {%- set rec = {'sched': s|upper,'time':start,'when':'today','zones':zns.z,'mins':zns.mins} -%}
        {%- if today_at(start) >= now() -%}{%- set ns.upcoming = ns.upcoming + [rec] -%}
        {%- else -%}{%- set ns.past = ns.past + [rec] -%}{%- endif -%}
      {%- endif -%}
      {%- if is_state('input_boolean.garden2_' ~ s ~ '_water_' ~ wd[(now()+timedelta(days=1)).weekday()], 'on') and (today_at(start)+timedelta(days=1)) <= now()+timedelta(hours=24) -%}
        {%- set ns.upcoming = ns.upcoming + [{'sched':s|upper,'time':start,'when':'tomorrow','zones':zns.z,'mins':zns.mins}] -%}
      {%- endif -%}
    {%- endif -%}
  {%- endif -%}
{%- endfor -%}
**Since midnight**
{% if winter %}- Garden system is in winter shutdown — nothing has run.
{% elif rain_off %}- Today's scheduled watering was **cancelled because of rain**{% if ns.past %}: {% for r in ns.past %}{{ r.time }} {{ r.zones|join(', ') }}{% if not loop.last %}; {% endif %}{% endfor %}{% endif %}.
{% elif ns.past|count > 0 %}{% for r in ns.past %}- {{ r.time }} (Schedule {{ r.sched }}): {{ r.zones|join(', ') }} — watered as scheduled.
{% endfor %}{% else %}- No scheduled watering was due earlier today.
{% endif %}
**Next 24 hours**
{% if winter %}- The garden system is in winter shutdown, so nothing will run.
{% elif ns.upcoming|count == 0 %}- No garden irrigation is scheduled in the next 24 hours.
{% else %}{% for r in ns.upcoming %}- **{{ r.when|capitalize }} at {{ r.time }}** (Schedule {{ r.sched }}): {{ r.zones|join(', ') }} — about {{ r.mins }} min total{% if r.when=='today' and rain_off %}, but **set to be SKIPPED** because of rain{% elif r.when=='tomorrow' %}{% if rn.tom < 0 %} (rain forecast unavailable){% elif rn.tom >= threshold %} — but rain is forecast ({{ rn.tom }}%, ≥{{ threshold }}% threshold), so **likely skipped**{% else %} — only {{ rn.tom }}% rain forecast, so expected to run{% endif %}{% endif %}.
{% endfor %}{% endif %}
```

## Instructions skeleton (if narrated as its own brief)

```
You are a concise home assistant giving a short garden-irrigation update. Use ONLY the facts below
(already computed — do not infer schedules or rain). Keep ONE line per scheduled run and NAME every
zone exactly as given (e.g. "LHS Upper Lawn (Blue)", "LHS Fence (Green&Yellow)") with its time and
minutes — do not merge or drop zones. Plain English, no markdown, no preamble. If a run was stopped
because of rain, say so plainly. If nothing is scheduled, say the garden has no watering planned in
the next 24 hours.

Facts:
<the rendered fact block above>
```

> **Tip:** because the fact block already emits clean per-run lines, the lowest-risk option for the
> daily summary is to drop the rendered block in **verbatim** under an *Irrigation* heading and let
> the LLM narrate the *other* sections — that guarantees no zone name is ever lost to paraphrasing.

## Verification (rendered via `ha_eval_template`, 2026-06-23)

**Live (both schedules disarmed today):**
```
- No further garden irrigation is scheduled in the next 24 hours.
```

**Scenario (now 08:00; Schedule A 04:00 was due earlier today; Schedule B 09:30 still to come; both
also run tomorrow; `garden_rain_cancel` = on; tomorrow's forecast 5% — the live Met Office figure
for 2026-06-24). Zone lists are the live valve config for each schedule:**
```
- Earlier irrigation today (04:00, Schedule A: LHS Upper Lawn (Blue) (10 min), LHS Lower Lawn (Green) (10 min)) was stopped because of rain.
- Today at 09:30 (Schedule B): LHS Fence (Green&Yellow) (5 min), RHS Fence (5 min) — about 10 min total, but this is set to be SKIPPED because of rain.
- Tomorrow at 04:00 (Schedule A): LHS Upper Lawn (Blue) (10 min), LHS Lower Lawn (Green) (10 min) — about 20 min total — only 5% rain forecast, so expected to run.
- Tomorrow at 09:30 (Schedule B): LHS Fence (Green&Yellow) (5 min), RHS Fence (5 min) — about 10 min total — only 5% rain forecast, so expected to run.
```

**Same scenario but tomorrow's forecast 70% (≥ threshold) — tomorrow's rows still name their valves:**
```
- Tomorrow at 04:00 (Schedule A): LHS Upper Lawn (Blue) (10 min), LHS Lower Lawn (Green) (10 min) — about 20 min total — but rain is forecast (70%, at/above the 40% threshold), so it will likely be skipped.
- Tomorrow at 09:30 (Schedule B): LHS Fence (Green&Yellow) (5 min), RHS Fence (5 min) — about 10 min total — but rain is forecast (70%, at/above the 40% threshold), so it will likely be skipped.
```

All requirements confirmed: zones to be watered, the time/duration, the rain-skip flag on **today's**
runs, the past-tense "stopped because of rain" for an earlier run, **and** a forecast-based verdict
on **tomorrow's** runs (which would otherwise carry no rain indication).

## Notes

- **Rain reason detail.** The auto-cancel automation distinguishes *rained-recently* vs *forecast*
  in its persistent notification, but that reason isn't stored in an entity — so this block states
  the user-requested "stopped because of rain" without re-deriving the cause. To surface the cause,
  have `garden_rain_auto_cancel_check` also write its reason to a new `input_text.garden_rain_reason`
  and add it to the fact block.
- **Today vs tomorrow rain logic differ — by design.** `garden_rain_cancel` is the *actual*
  decision but is cleared overnight, so it only applies to `when == 'today'` runs. Tomorrow's runs
  instead show a **forecast verdict** from `weather.get_forecasts` (daily precip probability vs the
  same `garden_rain_threshold`). It's a likelihood, not a commitment — the real cancel for tomorrow
  is still made ~30 min before that run, so a forecast can change by then.
- **Daily vs hourly forecast.** This block uses the **daily** precip probability for simplicity;
  `garden_rain_auto_cancel_check` uses the **hourly** max over its next-24 h window. They can differ
  slightly for a borderline day — acceptable for a "tomorrow outlook" line. Switch to `type: hourly`
  and take the max around each run's time if you want exact parity.
- **Veg trug** (`switch.tap_veg_trug`) is a valid slot target but currently has duration 0 in both
  schedules, so it never appears — by design (0 = unused slot).
- If the briefing's `input_text` mirror is added, keep the section short — `input_text` caps at 255
  chars (same constraint as the energy/charge briefs).
```
# Prompt — Evening Charge Advisory (Phase 2)

A nightly *"should I charge the Audi tonight?"* suggestion. The first piece that crosses from
**narration** (Phase 2) toward **optimisation/advice** (Phase 6) — but it stays **advisory only**:
HA computes the facts, the LLM phrases a suggestion, the human decides and acts. No automated
charging (that waits for Phase 6).

## Why this one works
Every input already exists in HA, and the one fuzzy step — *"does tomorrow's diary imply a long
drive?"* — is exactly where the 8B model adds value over a template, **and** it's safe to let it be
imperfect because the output is a suggestion to a person, not a control action (ADR-0001).

## How it's wired
- **Helper** `input_text.ai_charge_advisory` (255 chars) — holds the latest text for the dashboard.
- **Script** `script.charge_advisory_generate` — pulls tomorrow's diary (`calendar.get_events` on
  `calendar.richard` + `calendar.ruth`), builds the fact block, calls `ai_task.generate_data` on
  `ai_task.ollama_ai_task_llama3_1`, strips wrapping quotes, writes the text to the helper.
  Reusable + button-testable (same shape as `script.refresh_energy_snapshot`).
- **Automation** `automation.evening_charge_advisory` — `time` trigger at **21:00**, runs the
  script, then notifies **Slack `#home-assistant`** (`notify.basingbourne`) **and** mobile.

> **Jinja caveat (Phase 2 lesson):** in an automation/script the templates below render against live
> state automatically. In a direct service/API call they are NOT rendered — pre-render for ad-hoc tests.

## Decision skeleton (encoded in the prompt; HA supplies the facts)
```
home?  ── no ─→  say nothing about charging (just note it's away)
  │ yes
plugged in?  ── no ─→  "at X%, not plugged in — plug in tonight if you want a charge"
  │ yes
SoC < 40%        → charge tonight (high confidence)
SoC 40–60%       → only if tomorrow needs a long drive (judge from the diary)
SoC > 60%        → normally skip, unless the diary clearly shows a long trip
        └ overlay: strong solar tomorrow → could top up from solar by day, not grid overnight
```

## Entities used
| Fact | Entity |
|---|---|
| State of charge | `sensor.audi_q4_e_tron_state_of_charge` (%) |
| Target SoC | `sensor.audi_q4_e_tron_target_state_of_charge` |
| Plugged in | `binary_sensor.audi_q4_e_tron_plug_state` |
| At home | `device_tracker.audi_q4_e_tron_position` |
| Range | `sensor.audi_q4_e_tron_range` (km) |
| Tomorrow's solar | `sensor.energy_production_tomorrow` (Forecast.Solar — weather + daylight baked in) |
| Unit rate | `sensor.octopus_energy_electricity_21j0061481_2000017536930_current_rate` |
| Tomorrow's diary | `calendar.richard`, `calendar.ruth` |

## Instructions template (hardened)
```
You are a concise home-energy assistant giving an EVENING car-charging suggestion for tonight.
Use ONLY the facts below (already computed — do not do arithmetic or re-infer charge state).
Advisory only; the household decides. Reply in 1-2 short sentences, under 240 characters, plain
English, no markdown, no preamble.

Apply this guide:
- Car not at home -> say nothing about charging, just note it's away.
- At home but NOT plugged in -> main point is to plug in tonight if a charge is wanted.
- Battery under 40% -> suggest charging tonight.
- Battery 40-60% -> only worth charging if tomorrow needs a long drive (judge from the diary).
- Battery over 60% -> normally skip, unless the diary clearly shows a long trip.
- If tomorrow's solar is strong, you may note it could top up from solar by day instead of the grid.

Facts:
- Car location: home / away
- Plugged in: yes / no
- Battery (state of charge): NN% (band: low / medium / ample)
- Charge limit set on car: NN%
- Estimated range: NN km
- Tomorrow's solar forecast: N.N kWh
- Current electricity rate: GBP 0.NN/kWh
- Tomorrow's diary: <Who: summary (HH:MM) @ location | ...> or "nothing scheduled"
```

## Verification (test-fire, 2026-06-22, warm model)
Facts: home, **not plugged**, SoC **83 %** (ample), solar tomorrow ~9 kWh, diary = a 08:00 leisure-centre
appointment + a midday cleaner (both local). Output:
> "Since the Audi is already at home and not plugged in, it's a good idea to plug it in if you want to
> charge it overnight. However, with the battery currently at 83%, which is well above 60%, there
> doesn't seem to be an immediate need for charging unless tomorrow has a long drive planned — but your
> diary doesn't indicate anything particularly strenuous on the cards."

Correct call (no charge needed), correctly read the diary as "no long drive," respected the
not-plugged guard. (Prompt later tightened to ≤240 chars so it fits the `input_text` mirror.)

## Delivery (all live)
- **Slack** — `#home-assistant` via the existing **Slack integration** (title "Basingbourne",
  service `notify.basingbourne`, target `["#home-assistant"]`). Was already configured — no token
  work needed. Reused later for Phase 6 two-way control.
- **Mobile** — `notify.mobile_app_rich_iphone_15`.
- **Dashboard** — "Charge tonight?" section on `/ai-housekeeper` bound to `input_text.ai_charge_advisory`.

> **Jinja + `notify` trap:** templates render in the **automation** but NOT in a direct
> service/API call — an ad-hoc `notify.basingbourne` test posts the literal `{{ … }}`. Pre-render
> for manual tests; the automation renders fine.

## Notes
- `input_text` caps at 255 chars — the prompt asks for ≤240 and the script truncates defensively.
- Audi cloud polls periodically (there's an `…_api_requests_remaining` budget); the 21:00 read may be
  a little stale — acceptable for an evening "plan tonight" nudge.
- Sibling brief: the morning recap (`prompts/energy-snapshot.md` → scheduled). Same pipeline.

# Prompt — Energy Snapshot (Phase 2 MVP)

The first narration test, now proven working end-to-end (HA → AI Task → Ollama `llama3.1:8b`
→ text). Self-contained: uses only live entities that already exist.

## How it's wired
An automation (or manual run) calls `ai_task.generate_data` against the AI Task entity
`ai_task.ollama_ai_task_llama3_1`, passing the `instructions` below. The returned text is sent
to a `notify` target / Sonos TTS.

> **In an automation**, the Jinja below renders against live state automatically. **In a direct
> service/API call**, Jinja is NOT rendered — pre-render it (e.g. `ha_eval_template`) and pass the
> finished string.

## Instructions template (hardened)

```
You are a concise home energy assistant. Using ONLY the facts below (already computed — do NOT
reinterpret or infer import/export yourself), write a friendly 2-3 sentence snapshot of the
household's electricity right now. Plain English, no markdown, no preamble.

Facts:
- House load right now: {{ states('sensor.house_power_now') }} W
- Solar generating right now: {{ states('sensor.myenergi_harvi_11174172_power_ct_generation') }} W
- Grid right now: {% set g = states('sensor.myenergi_harvi_11174172_power_ct_grid') | float(0) %}{{ 'importing %d W from the grid' | format(g | round | int) if g >= 0 else 'exporting %d W to the grid' | format((g * -1) | round | int) }}
- Energy the house has used today: {{ states('sensor.house_energy_daily') }} kWh
- Solar generated today: {{ states('sensor.myenergi_hub_20111566_generated_today') }} kWh
```

## Lesson learned (important for all future prompts)
The first attempt fed the model a **signed** grid value with a "negative = exporting" hint, and
`llama3.1:8b` got it wrong — it claimed the house was exporting while it was importing. Fixing it
by **pre-computing the import/export direction in HA** (the template above) produced correct output.

**Rule:** never ask the 8B model to interpret signs, do arithmetic, or compare values — compute the
plain fact in HA and let the model only phrase it (ADR-0001). This is why prompts state derived
facts ("importing 1302 W"), not raw signed readings.

## Notes
- Model resident on the RTX 3090 (`keep_alive -1`); **first call after idle is a cold start** and
  can exceed the tool's response timeout — the result still completes, just re-fire once warm.
- Next prompt: `morning-briefing` (batteries, offline devices, water, weather, vs-average) —
  depends on roll-up helpers + the deferred statistics baselines.

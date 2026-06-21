# Prompt — Energy Snapshot (Phase 2 MVP)

The first narration test. Deliberately self-contained: uses only live entities that already
exist (no deferred statistics baselines), so it proves the HA → AI Task → notification pipeline
end-to-end with `llama3.1:8b`.

## How it's wired
An automation (or manual run) calls `ai_task.generate_data` against the Ollama **AI Task**
entity, passing the `instructions` below with live values injected via templates. The returned
text is sent to a `notify` target / Sonos TTS.

## Instructions template (Jinja-injected live values)

```
You are a concise home energy assistant. Using only the readings below, write a friendly
2-3 sentence snapshot of the household's electricity right now. State whether the house is
importing from or exporting to the grid, mention solar output, and note today's usage so far.
Plain English, no markdown, no preamble.

Readings:
- House load now: {{ states('sensor.house_power_now') }} W
- Solar generating: {{ states('sensor.myenergi_harvi_11174172_power_ct_generation') }} W
- Grid (negative = exporting): {{ states('sensor.myenergi_harvi_11174172_power_ct_grid') }} W
- House energy used today: {{ states('sensor.house_energy_daily') }} kWh
- Solar generated today: {{ states('sensor.myenergi_hub_20111566_generated_today') }} kWh
```

## Notes
- Keep `num_ctx` ≥ 8192 on the AI Task model config.
- This is narration only — all numbers are computed by HA (the analyst); the model just phrases
  them (ADR-0001).
- Next prompt to build: `morning-briefing` (adds batteries, offline devices, water, weather) —
  depends on a couple of roll-up helpers + the deferred statistics baselines for "vs average".
```

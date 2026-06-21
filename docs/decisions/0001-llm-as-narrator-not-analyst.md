# ADR-0001 — The LLM narrates; Home Assistant analyses

- **Status:** Accepted
- **Date:** 2026-06-21

## Context

The instinct is to "have the AI learn the house's consumption" by letting a model
watch everything continuously and work things out. We run a local Ollama model
(`qwen2.5:14b`) on an RTX 3090.

## Decision

Keep two roles strictly separate:
- **Home Assistant is the analyst.** All measuring, maths, anomaly detection, and
  history live in HA helpers (`statistics`, `utility_meter`, `integration`,
  `derivative`, `threshold`, `trend`) and the recorder DB.
- **The LLM is the narrator.** It only reasons over and explains data HA has already
  computed, on a schedule or on demand, plus fuzzy tasks (vision, NL control).

## Consequences

- **Accuracy:** deterministic helpers don't hallucinate figures; a 14B model asked to
  find anomalies in a number dump will.
- **Cost/power:** no need to run the GPU continuously; the model fires periodically.
- **Reproducibility:** the "understanding" is inspectable HA config, not opaque model state.
- **Implication for "AI builds helpers":** yes — but it's Claude/MCP (supervised,
  build-time) that creates helpers, not the local model autonomously. See ADR-0002.

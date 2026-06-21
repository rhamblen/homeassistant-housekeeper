# Architecture

## The core pattern: analyst vs narrator

Two roles, kept strictly separate:

- **Analyst = Home Assistant.** Helpers (`statistics`, `utility_meter`,
  `integration`, `derivative`, `threshold`, `trend`), the recorder/history DB, and
  long-term statistics do all measuring, maths, anomaly detection, and remembering.
  Deterministic, free, runs 24/7.
- **Narrator = the local LLM (Ollama).** Reasons over and explains what HA already
  captured, on a schedule or on demand. Also handles fuzzy perception (camera vision)
  and natural-language control.

We never hand the LLM a pile of numbers and ask it to find the anomaly — a local 14B
model is unreliable at that and costly to run continuously. See
[ADR-0001](decisions/0001-llm-as-narrator-not-analyst.md).

## Two AIs, two jobs

| | What | Role in this project |
|---|---|---|
| **Claude + MCP** (supervised) | Cloud model with tools | Builds the scaffolding — creates helpers, automations, prompts, docs, with human sign-off per batch |
| **Local Ollama agent** | `qwen2.5:14b` in HA | Day-to-day narration, Q&A, control |

The local model does **not** autonomously write config. Build-time changes are made
deliberately (by Claude/MCP or by hand), reviewed, and recorded in this repo.

## Delivery modes

- **Scheduled digests** — AI Task wakes on a timer, composes a report, sends to
  phone/Sonos. (morning briefing, energy review, water weekly)
- **Anomaly watchdogs** — HA detects a condition and fires the AI only when something
  is wrong. (leak, device offline, big load)

## Hardware & network

| Component | Where | Address / detail |
|---|---|---|
| Home Assistant | (HAOS/Supervised) | `192.168.1.64` — HA 2026.7 dev, 5,732 entities, 26 areas |
| Ollama | Docker on Unraid **UR1** | `http://192.168.1.32:11434` (br0, own IP) |
| GPU 0 | UR1 | RTX 3060, 12 GB → vision model |
| GPU 1 | UR1 | RTX 3090, 24 GB → 14B brain |
| Host | UR1 | Ryzen 7 5800X, 32 GB RAM |
| cloudflared | UR1 | `192.168.1.31` (rooroo.uk tunnel) |

**Notes**
- Both GPUs already carry baseline VRAM from other containers — usable headroom is
  roughly ~13 GB (3090) / ~6 GB (3060), not the full 36 GB.
- Ollama container restart policy was `no` — change to `unless-stopped`.
- Pin models with `CUDA_VISIBLE_DEVICES` so the 14B (3090) and vision (3060) don't contend.

## Model choices

| Use | Model | Fits |
|---|---|---|
| Brain (conversation + AI Task) | `qwen2.5:14b-instruct` (~9 GB) | RTX 3090; step up to `qwen2.5:32b` if freed |
| Vision (cameras) | `qwen2.5vl:7b` (~6 GB) or `moondream` (~1.7 GB) | RTX 3060 |

Set `num_ctx` to 8192+ for the conversation agent (default 2048 is too small once
entities are in the prompt).

## Data-source notes

- **Real-time power** comes from **myenergi** (harvi grid CT), not the smart meter.
  House load = `generation + grid_ct` (grid CT negative = exporting).
- **Octopus / smart meter** is half-hourly and ~a day delayed (no Home Mini present):
  use it for *cost/tariff*, not live demand.
- **Washing machine** (Samsung via SmartThings) reports cycle state (`machine_state`,
  `job_state`, `completion_time`) and cumulative `energy` (kWh) reliably; its
  instantaneous `power` (W) is coarse — use the energy *delta* for per-cycle cost.

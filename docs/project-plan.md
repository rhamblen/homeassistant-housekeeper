# Project Plan — AI Home Manager

> Phased roadmap. Each phase has a clear goal, what we build, prerequisites,
> deliverables, the rationale ("why"), and exit criteria. Status is tracked in
> the table below and updated as we go, so the repo always reflects reality.

**Last updated:** 2026-06-21

## Guiding principle

**Home Assistant is the analyst; the LLM is the narrator.**
HA's recorder + helpers do the measuring, maths, and remembering (deterministic,
free, always-on). The local LLM only *reasons over and explains* what HA has
already captured. We never ask the LLM to crunch raw numbers or detect anomalies
from a data dump — it is unreliable at that and expensive to run continuously.
See [ADR-0001](decisions/0001-llm-as-narrator-not-analyst.md).

## Phase summary

| Phase | Goal | Needs Ollama? | Status |
|------:|------|:-------------:|--------|
| 0 | Foundations: connect Ollama, scaffold repo, curate exposed entities | Set up | ☐ Not started |
| 1 | Consumption capture: helpers that record per-device energy history | No | ◐ In progress — Tier-1 helpers built (see [phases/phase-1](phases/phase-1-consumption-capture.md)) |
| 2 | AI narration: first scheduled reports (morning briefing, wash cost, energy review) | Yes | ☐ Not started |
| 3 | Anomaly watchdogs: silent until something is wrong | Yes | ☐ Not started |
| 4 | Load fingerprinting: learn what each device/room draws | Yes | ☐ Not started |
| 5 | Camera intelligence: vision-model smart notifications | Yes (vision) | ☐ Not started |
| 6 | Conversational control + energy optimisation | Yes | ☐ Not started |
| 7 | (Optional hardware) per-circuit / per-room metering | No | ☐ Not started |

Legend: ☐ Not started · ◐ In progress · ☑ Done

---

## Phase 0 — Foundations

**Objective:** Get the plumbing and project hygiene in place so every later phase
is reproducible and explainable.

**What we build**
- Connect the **Ollama** integration in HA → `http://192.168.1.32:11434`.
- Pull models on the GPU box: `qwen2.5:14b` (brain) and `qwen2.5vl:7b` (vision).
- Confirm `OLLAMA_HOST=0.0.0.0` and set the container restart policy to
  `unless-stopped` so it survives a UR1 reboot.
- This repository (scaffolded).
- Curate the Assist **exposed-entity** set (~40–80) so the conversational layer
  in Phase 6 has a clean, well-named surface (we have 5,732 entities — exposing
  all of them would break the LLM).
- Capture a **baseline inventory** of what is already metered ([inventory.md](inventory.md)).

**Prerequisites:** none.

**Deliverables:** working HA↔Ollama link; models pulled; repo committed; exposed-entity list documented.

**Why:** Everything downstream depends on a reachable model and a tidy, documented
starting point. Doing the housekeeping now is what lets us "come back and know what
we did and why."

**Exit criteria:** `ha_eval_template` / a test prompt round-trips through Ollama; repo has an initial commit.

---

## Phase 1 — Consumption capture foundation

**Objective:** Start *recording* structured per-device consumption history. This is
the "learning" — it lives in HA, not the LLM. No AI required.

**What we build** (all via proper HA helpers, per [ADR-0002](decisions/0002-repository-as-source-of-truth.md))
- `sensor.house_power_now` — live whole-house load (template: `generation + grid_ct`).
- **`utility_meter`** helpers (daily / weekly / monthly) on every already-metered device:
  washing machine, pool (Shelly PM), EV (zappi), immersion (iBoost), pool heat pump,
  whole-house (harvi).
- **`integration`** (Riemann sum) helpers to derive kWh for any power-only sensor.
- **`statistics`** helpers (7–30 day mean / stdev) to establish baselines.
- Wire the new energy sensors into the **Energy Dashboard**.

**Prerequisites:** none (does not need Ollama).

**Deliverables:** per-device daily/weekly/monthly consumption history accumulating in HA.

**Why:** You cannot explain or optimise what you have not measured. Utility meters +
statistics are the substrate the AI narrates later. Baselines here are what enable
"42% above normal for a Thursday" in Phase 3.

**Exit criteria:** each target device has a working utility_meter; baselines populating; `house_power_now` tracks live.

---

## Phase 2 — AI narration layer (first reports)

**Objective:** Connect the narrator and ship the first genuinely useful daily output.

**What we build**
- AI Task wired to the Ollama conversation/AI-Task subentry.
- **Morning briefing** (~07:00): overnight events, batteries, offline devices, water
  status, yesterday kWh vs average, weather, EV state. The umbrella report.
- **"Wash finished + cost"** report: SmartThings energy delta × Octopus rate.
- **Energy review**: yesterday's top consumers, cheap-rate vs peak split, iBoost diversion.
- Delivery via mobile `notify` and Sonos TTS.

**Prerequisites:** Phase 0 (Ollama) + Phase 1 (data to narrate).

**Deliverables:** at least one scheduled report landing on phone/Sonos, test-fired before scheduling.

**Why:** Proves the full HA→AI Task→delivery pipeline end-to-end on low-risk, high-value
output, and gives daily payback immediately.

**Exit criteria:** morning briefing fires on demand and on schedule with correct, readable content.

---

## Phase 3 — Anomaly watchdogs

**Objective:** Report *only when something is wrong*. HA detects; AI phrases the alert.

**What we build**
- **Garden leak / unexpected flow**: flow > threshold, all valves closed, for 10 min,
  tiered by duration/time (pressure-washer vs leak). (Already designed.)
- **Device went dark**: a camera / lock / alarm unavailable > X hours.
- **Left on / left open**: tap, garage, blind, heating overnight.
- **Big load started**: `house_power_now` jumps > 2 kW with no known cause.
- **Battery will fail soon**: trend helper on existing battery sensors.

**Prerequisites:** Phase 1 (helpers/baselines), Phase 2 (narration).

**Deliverables:** each watchdog as a HA automation + AI-narrated notification, with repeat-protection.

**Why:** This is the "home manager" value — proactive, quiet, only speaks when it matters.

**Exit criteria:** each watchdog verified against a simulated or real trigger.

---

## Phase 4 — Load fingerprinting / disaggregation

**Objective:** Build an understanding of what each device — and, by inference, each
area — draws, without per-circuit hardware.

**What we build**
- **Passive learning**: correlate device-state events (washer `job_state`, Neff
  Home Connect on/off, smart plugs) with `house_power_now` deltas → self-labelling
  load library.
- **Manual "appliance test" capture button**: for hand-operated loads (kettle, hob),
  a button captures the before/after window and labels it.
- **Room/category roll-ups**: template/group sensors summing metered devices per area.
- AI **live attribution**: "2 kW jump ≈ immersion fingerprint."

**Prerequisites:** Phases 1–3.

**Deliverables:** a documented load library; energy reports upgraded from "usage spiked"
to "the immersion ran 3 hrs because solar was low."

**Why:** Turns raw totals into explanation. Leans on data you already have
(Home Connect, SmartThings) so most of it builds itself over a few weeks.

**Exit criteria:** library covers the major loads; attribution demonstrably correct on a known event.

---

## Phase 5 — Camera intelligence

**Objective:** Replace "motion detected" with meaningful descriptions.

**What we build**
- `qwen2.5vl:7b` (or `moondream`) on the RTX 3060.
- On Reolink object-detect → snapshot → AI Task vision → notification
  ("courier left a parcel", "fox in the garden", known-vehicle recognition).
- Optional evening **camera day digest**.

**Prerequisites:** Phase 0 (vision model), Phase 2 (AI Task pattern).

**Deliverables:** at least one camera (the Drive) wired to AI-described notifications.

**Why:** High-impact, self-contained, and entirely local — frames never leave the LAN.

**Exit criteria:** Drive camera produces an accurate natural-language alert on a real event.

---

## Phase 6 — Conversational control + energy optimisation

**Objective:** Talk to the house, and let it advise on when to use energy.

**What we build**
- Ollama as the **Assist conversation agent** with control enabled (using the curated
  exposed-entity set from Phase 0).
- Optional **local voice**: Whisper (STT) + Piper (TTS) + openWakeWord add-ons →
  replaces the current cloud `google_translate` TTS.
- **Energy optimisation advice**: combine Octopus rates + solar/iBoost + EV state →
  "good time to run a wash", "charge the battery tonight?".

**Prerequisites:** Phases 0–4.

**Deliverables:** working conversational control; optional voice pipeline; run-it-now advisor.

**Why:** The interactive layer, built last because it depends on a curated entity set
and the data/understanding from earlier phases to give good answers.

**Exit criteria:** natural-language control works reliably on the exposed set; advisor gives sensible timing.

---

## Phase 7 — (Optional) per-circuit / per-room metering

**Objective:** Real per-room consumption, if software inference proves insufficient.

**What we build**
- CT-clamp energy monitor on the consumer unit (Shelly Pro 3EM or Emporia Vue).
- Per-circuit utility_meter + statistics, re-running Phase 1's pattern per circuit.

**Prerequisites:** decision after Phase 4 (is inference good enough?).

**Deliverables:** measured (not inferred) per-circuit/room energy.

**Why:** The only way to get genuine room-level truth; deferred until we know whether
inference is sufficient, to avoid buying hardware we don't need.

**Exit criteria:** per-circuit data flowing and documented.

---

## Open decisions

- **Repo visibility & name** before first push (default: public, `ha-ai-home-manager`).
- **License** (default: MIT — see `LICENSE`).
- **Phase 7 hardware** — defer decision to after Phase 4.
- **Local voice (Phase 6)** — optional; depends on appetite to replace Alexa/Sonos TTS.

# Jarvis

**English** · [简体中文](README.zh.md)

**A personal voice AI that grows with you.**

*Compounding memory, room-aware perception, and a self-improving skill loop.*

## Overview

Jarvis is an end-to-end voice assistant designed around a single thesis: assistant utility compounds. Most voice AIs today — Alexa, Siri, ChatGPT — treat each interaction as stateless. Jarvis inverts this: an observer extracts priority-tagged observations from each conversation, a stable-prefix builder injects them into the next session's prompt context, a trace table records every tool call for the skill-discovery loop. The longer it runs, the less you have to repeat yourself.

Built end-to-end without LangChain or any agent framework. 26 core modules, 1,245 tests, designed to run continuously on Mac (development) and Raspberry Pi 5 (production), with an Electron desktop pet on the side.

## Capabilities

### Full-duplex interrupt

During TTS playback, a dedicated mic thread gates audio through Silero VAD into per-utterance segments and dispatches each closed segment asynchronously to the same SenseVoice ASR used by the main loop. On VAD trigger, playback ducks to 30% volume over a 30ms ramp via a custom PortAudio stream player. On confirmed keyword match (`{"停", "等一下", "打住", "暂停", "等等", ...}`) the ring buffer flushes and the LLM cancels. Pre-roll (500ms) captures the initial consonant of single-character keywords; post-roll (200ms) catches trailing fricatives.

| Metric | Value |
|---|---|
| speech-to-detect ("停"), median | 1179 ms |
| speech-to-detect ("等一下"), median | 911 ms |
| p95 latency | < 1850 ms |
| False positives on 30s controlled silence | 0.0 / s |
| Audio underflows over 10818 callbacks | 0 |

In practice, mid-sentence "停" drops the volume within roughly 350ms of speech onset and reaches full stop within another 700ms — smooth fade, no restart artifacts, no inter-sentence gap, no swallowed first consonants. Long-term: insert silence at clause boundaries (the empirical ~80% of natural interrupts happen there) to bring perceived latency near zero, and add XMOS XVF3800 directional gating once hardware lands so the system ignores TV and family voices.

### Compounding memory

Each completed conversation triggers an observer (LLM function calling, Grok-4.20 primary / Gemini 2.5 Flash fallback) that extracts priority-tagged text bullets, grouped by date and stored in SQLite. A stable-prefix builder injects the relevant bullets into the next session's system prompt — prompt-cache-friendly, deterministic, no per-query vector retrieval on the read path.

| Module | Role |
|---|---|
| `memory/cold/observer.py` | Async extraction with four priority tiers (HIGH / MED / LOW / DONE) |
| `memory/hot/assembler.py` | Assembles personality + observations + recent turns into the LLM context, in cache-stable blocks |
| `memory/trace.py` | Per-turn analytics (path, tool calls, emotion, latency, outcome) for the skill-discovery loop |
| `memory/core/store.py` | SQLite, six tables: memories / user_profiles / episodes / episode_digests / memory_relations / observations |

A typical observation log:

```
Date: 2026-04-17
* [HIGH] (14:30) User prefers warm yellow (2700K) in living room
* [MED]  (15:12) User mentioned weekend trip to Vancouver to see friends
* [DONE] (15:45) Reminder set for coffee machine descaling
```

Eight extraction models were benchmarked across twenty Chinese home-dialogue fixtures (smart-home, preference, state-change, temporal, emotion, correction, multi-entity, completion). grok-4.1-fast took the highest hallucination-aware F1 at 0.91. The shipped primary is grok-4.20 (F1 0.88, $0.031 per 100 turns, p95 4.8s) — the cheapest of the eight, 1.6s faster at p50, and the 0.03 F1 gap falls inside the +/-3pp noise band at n=20, which is not worth paying for on a background cold path. Gemini 2.5 Flash was the only model with zero hallucinations, at twice the cost, and is kept as the fallback. In practice, mention something next week that came up today and it is already part of the prompt context — no "I don't have access to previous conversations" wall, no manual replay.

### Self-improving skill loop

Skills register through a unified `tool_registry` in two formats: Python `@jarvis_tool` decorators for things that need code (12 live functions across `tools/reminders.py`, `tools/smart_home.py`, `tools/time_utils.py`, `tools/todos.py`) and YAML declarative specs for HTTP-wrapper-style skills (`skills/weather.yaml` plus auto-migrated `skills/learned/exchange_rate.yaml`). Both surface to the LLM as identical OpenAI-compatible function-calling schemas. Annotations (`read_only`, `destructive`, `idempotent`, `required_role`) gate each tool through a four-tier RBAC hierarchy: guest < family < trusted < owner.

```yaml
name: get_weather
parameters:
  - {name: city, type: string, default: Victoria}
action:
  type: http_get
  url: "https://wttr.in/{{ city }}?format=j1"
  retry: {max: 3, delay_ms: 1000, backoff: exponential}
response:
  template: "{{ city }} weather: {{ desc }}, {{ temp_c }}C..."
security:
  allowed_domains: [wttr.in]
```

YAML actions execute through a Jinja2 sandbox with per-skill domain whitelisting and an RFC1918 loopback block for SSRF protection. The discovery loop builds on the trace table that's already in place: a nightly batch will detect hot-spot intent patterns (frequency + importance + user-correction signal), draft new YAML candidates from 3-5 representative examples, run a 7-day shadow period with three-tier output similarity judging (structural / embedding / LLM-as-judge), and promote through canary monitoring with auto-rollback on regression. The static layer is live; the discovery pipeline lands incrementally on the same registry, so new skills appear without restarting the assistant.

### Multi-tier LLM resilience

Main response generation runs on OpenAI presets switchable at runtime by voice — `gpt-5.4-mini` (fast) and `gpt-5.5` (deep). Observation extraction runs on xAI grok-4.20 with Gemini 2.5 Flash as fallback. Ahead of any cloud call, `core/regex_router.py` short-circuits ~17 strictly anchored `^...$` patterns straight to a local tool in 0ms; every miss falls through to the LLM. The earlier LLM-based intent router was deleted after it measured 73.9% accuracy on 80 real traces — a regex layer that refuses to guess beat a 70B model that did. Every external call sits behind a circuit breaker (HEALTHY → DEGRADED → UNAVAILABLE) and falls through deterministically.

### Role-based device permissions

A four-tier permission model (guest → family → trusted → owner) gates smart-home device control: each device declares a `required_role` and the runtime denies actions below that tier. Currently single-user — the active role is hardcoded to `owner` — but the plumbing supports future identification backends.

## Architecture

```
   Mic ─→ Wake Word ─→ Record (VAD-gated)
                          │
                          ↓
                 SenseVoice ASR
                          │
                          ↓
            RegexRouter  (~17 anchored patterns, 0ms — hit: local tool)
                          │
                          ↓  (miss)
            Cloud LLM (streaming + tool-use loop)
                          │
                          ↓
            TTS pipeline (MiniMax WS → MiniMax HTTP)
                          │
                          ↓
            AudioStreamPlayer (sample-accurate gain ducking)
                          │
                          ↓
                       Speaker

   Background:    Observer extracts observations → SQLite
                  Trace records every turn for the skill-discovery loop

   During TTS:    Mic → VAD-gated segments → shared SenseVoice path
                       → keyword match → soft duck (30ms) or hard stop
```

## Hardware roadmap — spatial intelligence

The next iteration replaces the off-the-shelf USB microphone with an [XMOS XVF3800](https://www.xmos.com/xvf3800/) reference board. The chip provides direction-of-arrival, beamforming, distance estimation, and reverberation fingerprinting in hardware — turning Jarvis from an audio device into a spatial agent.

Concretely, this enables:

- **Room-aware control.** "Open the lights" without specifying which room — direction-of-arrival + acoustic fingerprint identify the space.
- **Zone-based personas.** Different tone, wake-word policy, and TTS volume by location (desk / sofa / bedroom / kitchen).
- **Distance-adaptive TTS.** Whisper at 0.5m, project at 3m, automatic.
- **Follow mode.** No-wake-word continuous conversation, gated by direction-of-arrival to suppress false triggers from TV or other speakers.
- **Cross-room handoff.** With multiple devices, the conversation follows you between rooms.

None of this is built — the section describes the intended next iteration, not shipped behaviour.

## Tech stack

| Layer | Stack |
|-------|-------|
| Wake word | openwakeword (`hey_jarvis_v0.1`) |
| ASR | SenseVoice-Small INT8 via sherpa-onnx · mlx-whisper fallback (Apple Silicon) |
| VAD | Silero VAD (ONNX), `headphones` / `speakers` mode-based thresholds |
| Fast path | `RegexRouter` — ~17 anchored patterns, 0ms, falls through to the LLM on miss |
| LLM | OpenAI `gpt-5.4-mini` (fast) / `gpt-5.5` (deep), switchable at runtime |
| Memory | Structured observation stream on SQLite · function-calling extraction (xAI grok-4.20 / Gemini 2.5 Flash) · stable-prefix injection |
| TTS | MiniMax WebSocket streaming → MiniMax HTTP fallback |
| Audio I/O | sounddevice + custom `AudioStreamPlayer` (PortAudio callback + ring buffer) |
| Devices | Philips Hue (live) · MQTT · in-memory sim |
| Desktop | Electron Pet Mode + Cmd+Space command panel |
| Spatial (next) | XMOS XVF3800 |

## Getting started

```bash
git clone https://github.com/samsara0xgg/Jarvis.git && cd Jarvis
uv pip install -r requirements.txt

# SenseVoice INT8 model (~228MB)
cd data
wget https://github.com/k2-fsa/sherpa-onnx/releases/download/asr-models/sherpa-onnx-sense-voice-zh-en-ja-ko-yue-int8-2024-07-17.tar.bz2
tar xf sherpa-onnx-sense-voice-zh-en-ja-ko-yue-int8-2024-07-17.tar.bz2
mv sherpa-onnx-sense-voice-zh-en-ja-ko-yue-int8-2024-07-17 sensevoice-small-int8
cd ..

# Minimum required env vars (config.yaml holds no secrets)
export OPENAI_API_KEY=...  # Main response LLM (gpt-5.4-mini / gpt-5.5)
export XAI_API_KEY=...     # Observation extraction (grok-4.20)

python jarvis.py --no-wake     # Development: press Enter to talk
python jarvis.py               # Production: wake word "Hey Jarvis"
```

Optional keys: `GEMINI_API_KEY` (observer fallback), `MINIMAX_API_KEY` and `MINIMAX_FALLBACK_API_KEY` (TTS — without them the assistant runs mute).

For the desktop pet:

```bash
python -m ui.web.server         # Terminal 1 — backend
cd desktop && npm start          # Terminal 2 — Electron
```

## Project structure

```
jarvis/
├── jarvis.py                   # Entry point — initializes all subsystems
├── config.yaml                 # Unified config (no secrets — env vars only)
├── core/                       # 26 modules — voice, ASR, LLM, TTS, interrupt, VAD
├── memory/                     # 14 modules — cold/ (observer, NLI, pricing), hot/ (assembler,
│                               #   conversation), core/store.py, manager, trace
├── auth/                       # Role-based device permission checks
├── devices/                    # Smart home backends (Hue / MQTT / sim)
├── desktop/                    # Electron Pet Mode + Cmd+Space command panel
├── ui/                         # Live2D web server + OLED display
├── skills/                     # YAML skills + learned/ runtime-generated skills
├── tools/                      # Built-in tool modules (reminders, smart-home, etc.)
├── system_tests/               # End-to-end runner (interactive + Claude Code mode)
├── tests/                      # 1,245 tests
├── deploy/                     # Raspberry Pi systemd + install scripts
├── esp32/                      # MicroPython firmware (sensor + relay nodes)
└── docs/                       # Design specs + git workflow
```

## Documentation

| Topic | File |
|-------|------|
| Git workflow + commit conventions | [`docs/git-guide.md`](docs/git-guide.md) |
| Composite skill interface | [`docs/architecture/composite-skill-interface-v1.md`](docs/architecture/composite-skill-interface-v1.md) |
| Skill lifecycle review | [`docs/architecture/skill-lifecycle-review-v1.md`](docs/architecture/skill-lifecycle-review-v1.md) |

The design notes behind the voice pipeline, the interrupt ASR migration, the XVF3800 research and the AudioStreamPlayer benchmark are kept in a private notebook and are not published in this repository.

## Tests

```bash
python -m pytest tests/ -q                     # 1,245 tests, ~25s, no API keys needed
python system_tests/runner.py --mode cc        # End-to-end (Claude Code)
python system_tests/runner.py                  # End-to-end (interactive)
```

## License

MIT — see [`LICENSE`](LICENSE).

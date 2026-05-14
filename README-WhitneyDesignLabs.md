# WireClaw — WhitneyDesignLabs Fork

A fork of [M64GitHub/WireClaw](https://github.com/M64GitHub/WireClaw) with refinements aimed at running WireClaw on **local Ollama servers with 8 GB-class GPUs** (e.g. GTX 1080) — and an optional baked-model package designed to pair with it.

This README covers the fork-specific delta. For the upstream firmware overview, hardware support, captive-portal setup, and tool catalog, see [README.md](README.md).

---

## What this fork adds over upstream

Five firmware-side improvements plus one optional baked model:

1. **P02-redesign — compact 4000-byte system prompt** ([data/system_prompt.txt](data/system_prompt.txt))
   Rewrite of the system prompt to fit under WireClaw's 4095-byte chip-truncation ceiling while preserving tool-call guidance. Empirical: small models (8B-class) tool-call more reliably with ruthlessly compact prompts than with longer "more helpful" instructions.

2. **P03-redesign — selective tool example augmentation** ([src/tools.cpp](src/tools.cpp))
   Three tools (`rule_create`, `chain_create`, `led_set`) carry inline JSON examples in their `description` field. Other tools stay stock. Selectivity matters: blanket augmentation hurts small models (drowns prompt context), targeted augmentation helps the specific failure modes we measured.

3. **P01 (v1+v2) — prose-leaked tool call detector** ([src/llm_client.cpp](src/llm_client.cpp))
   `parseToolCalls()` historically silently dropped tool calls if the model emitted them in prose form (XML-fenced, code-fenced, or naked JSON in `content`) instead of the proper `tool_calls` field. P01 detects three leak patterns and surfaces them via `m_error` so the operator can see what happened. Includes a boot-time self-test with four cases.

4. **P11 — `use_modelfile_system` config flag** ([src/llm_client.cpp](src/llm_client.cpp), [src/main.cpp](src/main.cpp))
   New boolean in `config.json`. When true, WireClaw omits any `role:"system"` message from the outbound `messages` array, letting a Modelfile-baked SYSTEM directive apply on the Ollama side. Default false (preserves stock behavior). This patch is what makes the bake usable through WireClaw — without it, baked SYSTEM is silently overridden by the client-side system prompt.

5. **P06 — `temperature` and `max_tokens` from config** ([src/llm_client.cpp](src/llm_client.cpp))
   Two existing `config.json` fields the upstream documented but didn't actually wire to the request body. Now they reach `buildRequest()`. Most useful for local-LLM operators who want `temperature: 0.1-0.3` for tool-calling reliability.

Plus the **baked Modelfile** at [bake/wireclaw-agent-v1.Modelfile](bake/wireclaw-agent-v1.Modelfile) — see [bake/README.md](bake/README.md) for the build, validation battery, and configuration.

---

## Two operating modes

### Stock mode (recommended for first-time users)

`cfg_use_modelfile_system = false` (default in the upstream-PR-clean `p11-use-modelfile-system` branch). Works with any chat-capable LLM:

- Stock Ollama models (`llama3.1:8b`, `qwen3:8b`, etc.)
- OpenRouter / OpenAI / cloud LLM endpoints
- Any model the operator already has running

The fork's other improvements (P02, P03, P01, P06) still apply. This is the right mode if you don't want to run your own Ollama or build a baked model.

### Baked mode (recommended for 8 GB-class local-LLM operators)

`cfg_use_modelfile_system = true` and `cfg_model = "wireclaw-agent:v1"`. WireClaw skips the client-side system message; a Modelfile-baked `wireclaw-agent:v1` model on your Ollama server provides the SYSTEM directive instead.

Build the bake: see [bake/README.md](bake/README.md).

The `wdl-v1` branch defaults to baked mode. Stock-mode operators on `wdl-v1` need to override either via `config.json` on LittleFS or the captive-portal web UI.

### When to use which

| Use case | Mode |
|---|---|
| First-time user, just trying WireClaw out | **Stock** |
| OpenRouter / OpenAI / cloud LLM | **Stock** |
| You don't want to run Ollama | **Stock** |
| Local Ollama, want measurably better tool correctness on time-based-rules + chain_create + compound memory recall | **Baked** |
| Production deployment where false action narrations would be costly | **Stock** (see "Known limits" below) |

The empirical evidence for baked-mode improvements is in [bench/](bench/) — direct-curl T8 (clock_hhmm time-based rule) and T9 (compound favorite-color) both pass on the bake but fail on stock `llama3.1:8b` across every prompt iteration tested. See `sync/worklog.md` Phase 2A entry (2026-05-13) for the 9/9 curl battery results.

---

## Known limits of the bake

Be honest about what the bake fixes and what it doesn't.

**The bake fixes:**
- Tool selection on time-based rules (`clock_hhmm`, `condition:"eq"`, `threshold:HHMM`)
- Tool selection on periodic rules (`condition:"always"`, `interval_seconds:N`)
- Compound memory-recall workflows (`file_read` then act on the value)
- Identity stability and constitutional refusals (cites SOUL.md article when refusing)
- Conversational default (model doesn't auto-fire `device_info` on greetings)

**The bake does NOT fix** (Phase 2B chip integration measurements, 2026-05-13):
- **Pseudo-prose wrap-ups.** After a tool call returns, the model's wrap-up text sometimes leaks Python-call-syntax (e.g. `(I called the temperature_read tool and it returned 27.0.)`). Aesthetic; the action fires correctly.
- **Fabricated action narrations.** More serious: model wrap-ups can claim an action that didn't fire. After `file_read` returns memory content, the model may emit "The LED is now purple." without ever firing `led_set`. Article 2 (Truth) violation. The chip's history mechanism then propagates the fabrication forward as established conversation context.
- **Tool-name collision on parallel multi-tool calls.** When the model intends `file_read` then `led_set` in one response, the second tool's function-name slot can collapse onto the first. The intended `led_set` becomes `file_read({"r":128,"g":0,"b":128})` — gets rejected with missing-path, no LED set, but the wrap-up confidently narrates success.

These three are weight-level concerns the bake's SYSTEM directive cannot fully fix. The fix path is a LoRA fine-tune (Phase 3 work, queued — see `bench/fork/lora/PHASE3.md` if present).

**Partial mitigation: `cfg_use_modelfile_system=false` (stock mode)** sidesteps the fabrication risk because the system prompt's prescriptive style somewhat constrains the model's wrap-up patterns. Trade-off: lose the bake's tool-correctness wins.

**Future option (sketched, not built): the P12 firmware patch** would post-check each agentic-loop iteration's wrap-up text against the tool-calls fired and flag/intercept claims that don't match. Guardrail, not fix. See `sync/worklog.md` 2026-05-13 entry if you want to take a swing at building it.

---

## Bench harness

The fork ships a Python tool-calling test suite at [bench/](bench/) that reproduces WireClaw's exact request shape against any Ollama (or OpenRouter) endpoint. 22 test cases covering tool selection, argument correctness, and four failure modes (prose leak, arg truncation, XML format, drown). Used to:

- Score model candidates (`llama3.1:8b` won at 19/22 over qwen3:8b at 19/22 with 5x speed; see `bench/results/run-*.md`)
- Validate that prompt + tool changes don't regress (the bench is the safety net before reflashing)
- Calibrate the bake (Phase 2A 9/9 results)

Bench measures tool-call structure and argument correctness only. Wrap-up coherence is qualitatively assessed via chip-side smoke testing — see `sync/worklog.md`'s chip-integration entries for the rubric.

---

## Getting started

### Stock mode (any model)

1. Clone this fork: `git clone https://github.com/WhitneyDesignLabs/WireClaw.git`
2. `cd WireClaw && pio run -e esp32-c6 --target upload --upload-port COMxx`
3. Connect to the captive portal on first boot, configure WiFi + Telegram + LLM endpoint via web UI

You're now running on `main` (upstream parity). To pick up the fork's improvements without the bake, check out one of the upstream-PR-clean branches:

- `p05-serial-send-clarification` — single-patch P05
- `p01-prose-tool-call-leak-detector` — single-patch P01 (v1+v2 squashed)
- `p11-use-modelfile-system` — single-patch P11 (default false)

Or use the integrated `wdl-v1` package branch — but flip `cfg_use_modelfile_system` back to false in `config.json` if you want stock mode.

### Baked mode (the WhitneyDesignLabs experience)

1. Clone the fork
2. **`git checkout wdl-v1`** — the integrated package branch
3. `pio run -e esp32-c6 --target upload --upload-port COMxx` — flash the package firmware
4. On your Ollama server: `ollama create wireclaw-agent:v1 -f bake/wireclaw-agent-v1.Modelfile` (see [bake/README.md](bake/README.md))
5. Captive-portal setup: WiFi + Telegram + `api_base_url = http://YOUR_OLLAMA_HOST:11434/v1/chat/completions`. The model and system-message-skip default to baked mode out of the box.

Optional pre-deploy validation: run the 9-test direct-curl battery against your built `wireclaw-agent:v1` (script and expected results in [bake/README.md](bake/README.md)).

---

## Acknowledgments

- **Upstream:** [M64GitHub/WireClaw](https://github.com/M64GitHub/WireClaw) by Mario Schallner — MIT licensed. All firmware infrastructure (HAL, captive portal, NATS, Telegram bridge, rule engine, web config, agentic loop) is upstream's work. The fork adds incremental improvements on top.
- **Project Opengates** and the **SAP-era constitutional bake methodology** (`baking-constitutional-models-8gb-vram.md` reproduction guide) — the foundation for the SOUL.md framework and the bake recipe shape.
- **Whitney Design Labs** — fork maintenance and the WireClaw-shaped adaptations.

## License

Inherits MIT from upstream. The compiled `wireclaw-agent:v1` model is additionally governed by Meta's [Llama 3.1 Community License](https://www.llama.com/llama3_1/license/) — see [bake/README.md](bake/README.md) for redistribution guidance.

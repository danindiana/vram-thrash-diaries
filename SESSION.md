# Session 1789341581 — Hermes Agent context/model fixes, Ollama thrashing fix, tool-call bridge fix

## Objective
Started from a context-window error switching Hermes Agent's chat model to
`dler-r1-7b:latest` (32,768 < Hermes's 64K minimum), then expanded into a
broader look at Hermes Agent's Ollama model management as issues surfaced.

## What was checked / done, in order

### 1. Context-window fix
`dler-r1-7b:latest` is Qwen2, architecture ceiling 131,072 — the 32,768 was
just the tag's baked-in `num_ctx`, not a hardware limit. Created
`dler-r1-7b-64k:latest` (derived Modelfile, `num_ctx=65536`). Updated
`~/.hermes/config.yaml` (`model.default`, `model.context_length`) and
`~/.hermes/context_length_cache.yaml` to match. **Note:** this was a bugfix
of convenience, never a considered chat-model choice — flagged as such in
(2) below.

### 2. Explained `memory-lab` profile and default-model history
`memory-lab` is a sandboxed profile (`write_approval: true`, separate Qdrant
store) originally for validating `holographic` memory; both `default` and
`memory-lab` now run **mem0 (OSS)** as of 2026-09-13 (session_1789338587/
1789339715). Memory config (mem0's LLM role, embedder, vector store) is
fully decoupled from `model.default` — swapping chat models never requires
touching memory config.

Researched whether `hermes3:8b` (the prior, unapplied recommendation from
session_1789339399) is outdated, and whether `ornith-1.5:9b` is a real,
better alternative. It is real (Qwen3.5/Gemma4-based, MIT, self-improving
RL training) with strong published benchmarks, but an independent hands-on
test found it failing badly on exactly the open-ended, multi-step
tool-calling tasks that dominate this Hermes Agent's workload. Recommended
trialing it in a sandbox profile against real logged tasks before touching
the default, rather than swapping on benchmark numbers alone. Output:
`hermes-default-model-ornith-vs-hermes3.md` (opened in Lite XL for the
operator).

### 3. Found and fixed Ollama model-loading thrashing
`ollama ps`/`journalctl -u ollama` investigation (prompted by the operator
noticing nemotron seemed to "elastically occupy" resources) found active
eviction thrashing: `OLLAMA_MAX_LOADED_MODELS=2` (set redundantly in two
overlapping systemd drop-ins, `local-only.conf` + `override.conf`) against
a workload needing up to 3 concurrent models per turn (chat model + mem0's
hardcoded `nemotron-3.5-lightning:1m` extraction LLM + `nomic-embed-text`
embedder). Measured: nemotron loaded/evicted 313 times, `nomic-embed-text`
256 times, in a 3-hour window — roughly every 30-90 seconds. A separate,
unrecognized `OLLAMA_MAX_VRAM=0` env var (not in this Ollama version's
`serve --help`) was masking the symptom as CPU/GPU splits rather than
errors. Output: `ollama-model-thrashing-nemotron.md` (opened in Lite XL).

Operator then asked about capping to one large model + reserving GPU
headroom for the desktop — confirmed both are real Ollama 0.33.3 knobs
(`OLLAMA_MAX_LOADED_MODELS`, `OLLAMA_GPU_OVERHEAD`) and applied them:

- Disabled `local-only.conf` (renamed with `.disabled-1789342323` suffix,
  original also kept as `.superseded-1789342323`).
- Rewrote `override.conf` as the single consolidated drop-in: removed the
  non-functional `OLLAMA_MAX_VRAM`, set `OLLAMA_MAX_LOADED_MODELS=1`, added
  `OLLAMA_GPU_OVERHEAD=1073741824` (1GiB/GPU reserved for desktop/GUI).
  `context.conf` (context length) left untouched — separate concern.
- `sudo systemctl daemon-reload && sudo systemctl restart ollama`. Verified
  post-restart: `journalctl` shows clean `"loaded runners" count=1` with no
  further eviction/"make room" messages — thrashing confirmed stopped.
- **Side effect acknowledged:** the restart dropped the live hermes
  session's (PID 1546635) in-flight model stream (`Connection refused` /
  stream-drop at 18:32:08); it auto-retried and recovered on its own.

### 4. Diagnosed live-session tool-calling errors
Checked `~/.hermes/logs/{agent.log,errors.log}` for the operator's reported
tool errors. Found and corrected a wrong initial diagnosis: a Qdrant
`.lock` file at `~/.hermes/mem0_qdrant/.lock` looked orphaned on a first
(wrong) check (`fuser` against the directory, not the file), but was
actually held open by the live hermes process. Deleted it anyway per
operator request; verified after the fact that this was harmless (Linux
unlink-while-open semantics — the process's existing fd is unaffected, no
new errors, lock not recreated) but was **not** an actual fix for anything,
since mem0 was already initialized and working.

The real cause of the recurring errors: the live session's model
(`qwen3:4b-thinking-2507-q8_0`, and before it `hermes3:8b-llama3.1-q5_K_M`
which failed outright with `"does not support thinking"`) was repeatedly
malforming Hermes's `tool_call` bridge — the deferred-tool batching layer
that requires `{"calls": [{"name","arguments"}]}` — and separately tried
routing the core (never-deferred) `mem0_search` tool through that same
bridge. Traced this to `hermes-agent/tools/tool_search_validation.py`
(`normalize_tool_call_entries`) and the `tools.tool_search` config in
`hermes_cli/config_defaults.py`.

**Fix applied:** added to `~/.hermes/config.yaml`:
```yaml
tools:
  tool_search:
    enabled: "off"
```
This removes the `tool_call` bridge entirely — deferrable tools go straight
into the model-facing tool list instead of behind a batching wrapper the
small model couldn't reliably use. Confirmed the edit parses correctly
(`yaml.safe_load`). Also flagged an unrelated, separate issue found in the
same logs: OpenRouter/Nous auxiliary providers marked unhealthy for
"payment/credit error" — a billing matter, not something fixed here.

### 5. `OLLAMA_NUM_PARALLEL=1` — measured, real win
Operator asked to try cutting `qwen3:4b-thinking-2507-q8_0`'s context to
fit more into VRAM; recommended `OLLAMA_NUM_PARALLEL=2→1` instead as the
bigger lever (halves reserved KV-cache slots). Applied via `override.conf`
+ restart. Measured before/after on the same model: SIZE 45GB→26GB,
CPU/GPU split 48%/52%→**5%/95%**. Confirmed real, not just theoretical.
Tradeoff noted: concurrent requests now queue instead of running in
parallel (irrelevant for this single-interactive-session usage pattern).

### 6. Diagnosed the real nemotron thrashing mechanism, fixed at the source
Operator hypothesized nemotron loads on every model completion, "possibly
coincident to memory compression." Confirmed precisely via `journalctl`
correlated with `agent.log`: mem0's extraction LLM (hardcoded to
`nemotron-3.5-lightning:1m`) fires a short, separate inference call on
session/turn boundaries (`CLI cleanup calling memory shutdown for session
... with N message(s)`), and with `OLLAMA_MAX_LOADED_MODELS=1` (from step
3) every one of those calls forces a full evict-reload cycle of whatever
chat model was active — reproducing thrashing symptoms with *any* chat
model, exactly as the operator suspected.

**Operator's fix direction: make mem0's extraction model run entirely on
CPU/system RAM**, not just "small," so it structurally can't compete for
GPU/VRAM at all. Built `mem0-extractor-cpu:latest` — derived from
`qwen3:4b-instruct-2507-q8_0` (6.9GB, already pulled) via Modelfile with
`PARAMETER num_gpu 0` (forces 100% CPU — confirmed via `ollama ps`
`PROCESSOR: 100% CPU`) and `PARAMETER num_ctx 16384` (extraction doesn't
need the model's full 262K context). System has 176GB free RAM, so this
costs nothing that matters. Updated `oss.llm.config.model` in both
`~/.hermes/mem0.json` and `~/.hermes/profiles/memory-lab/mem0.json`.

Operator then floated an alternative: have mem0 reuse whichever chat model
is currently loaded, avoiding a second model entirely. Checked
`hermes-agent/plugins/memory/mem0/*.py` — **no such mechanism exists**;
`mem0.json`'s model field is fully static, unconnected to `model.default`
or session-only overrides. Recommended against building this even if
possible: extraction quality would ride on whatever chat model is being
experimented with that day (this session alone cycled through 4 different
chat models, one of which failed outright and another which malformed
tool calls), and "thinking" models add reasoning overhead to every
extraction call. Revisit only once a chat model is stable and unchanging.

**Follow-on regression, caught by the operator immediately:**
`OLLAMA_MAX_LOADED_MODELS=1` counts every runner against one global slot
*regardless of device* — confirmed directly (`mem0-extractor-cpu` went to
`"Stopping..."` when a GPU chat-model request came in, despite using 0
VRAM). This reintroduced eviction thrashing, just between the chat model
and the CPU extractor instead of nemotron — matching the operator's "GPU
idle, model not running on it" report exactly (they were catching the
moment the CPU-only extractor held the one slot, having just evicted the
GPU chat model to get it). Fix: raised `OLLAMA_MAX_LOADED_MODELS` to `3`.
Verified all three roles resident simultaneously with zero eviction:
`qwen3:4b-thinking` (95% GPU), `nomic-embed-text` (78% GPU, 397MB),
`mem0-extractor-cpu` (100% CPU) — `ollama ps` showed all three at once,
free VRAM ~1GB/GPU (right at the `OLLAMA_GPU_OVERHEAD` reservation, so
desktop headroom held).

### 7. Found and disabled an unrelated, previously-unknown stray daemon
While diagnosing (6), discovered `ollama-gpu1.service` — a second,
completely separate Ollama systemd unit (port 11435), enabled at boot,
running since 12:55 today (~6.5hrs by the time found), described as
targeting a "Quadro M4000" GPU. `nvidia-smi -L` confirms only 2 physical
GPUs exist (RTX 4080 SUPER, RTX 5070) — the Quadro is gone, so this
service's `CUDA_VISIBLE_DEVICES=1` now silently resolves to the RTX 5070,
the same physical GPU the real `ollama.service` manages. It had nothing
loaded when found (not the direct cause of (6)'s symptom), but was a live
landmine: stale config, wrong GPU description, boot-enabled, totally
undocumented, capable of double-booking VRAM on the RTX 5070 any time
something targets port 11435. Operator confirmed: disable it.
`systemctl stop` + `disable` (unit file left in place, not deleted, for
reversibility).

## Status / what's NOT yet done
- **None of this round's config changes have taken effect on the live
  hermes session** (PID 1546635) — config loads at hermes startup. Needs a
  restart of that session to pick up: `tools.tool_search: off`,
  `OLLAMA_NUM_PARALLEL=1`, `OLLAMA_MAX_LOADED_MODELS=3`,
  `mem0-extractor-cpu`. Not restarted yet — didn't want to kill the
  operator's live session/conversation state without asking.
- `model.default` in `config.yaml` is still `dler-r1-7b-64k:latest` — a
  bugfix-of-convenience, not a considered choice. The live session is
  actually running `qwen3:4b-thinking-2507-q8_0` via a session-only
  override, independent of the config default. Neither has been decided on
  as the real long-term default; see `hermes-default-model-ornith-vs-hermes3.md`
  for that open question.
- Ornith-1.5-9B has not been pulled or trialed locally — recommendation
  only.
- OpenRouter/Nous credit issue is unresolved (out of scope — account/billing).

## Delivered to human operator
Both reports opened directly in separate Lite XL instances on the active
local display for direct viewing, independent of this CLI session:
- `hermes-default-model-ornith-vs-hermes3.md`
- `ollama-model-thrashing-nemotron.md`

## Files touched this session
- `~/.hermes/config.yaml` (model default/context, `tools.tool_search.enabled`)
- `~/.hermes/context_length_cache.yaml`
- `~/.hermes/mem0.json` and `~/.hermes/profiles/memory-lab/mem0.json`
  (`oss.llm.config.model` → `mem0-extractor-cpu:latest`)
- `/etc/systemd/system/ollama.service.d/override.conf` (rewritten, then
  `OLLAMA_NUM_PARALLEL` 2→1, `OLLAMA_MAX_LOADED_MODELS` 2→1→3)
- `/etc/systemd/system/ollama.service.d/local-only.conf` (disabled, backed up)
- `ollama-gpu1.service` — stopped and disabled (unit file kept, not deleted)
- Deleted `~/.hermes/mem0_qdrant/.lock` (verified harmless, not an actual fix)
- New Ollama models: `dler-r1-7b-64k:latest`, `mem0-extractor-cpu:latest`

# Ollama model thrashing: nemotron-3.5-lightning:1m and friends

Date: 2026-09-13
Scope: Ollama daemon on the-box (`ollama.service`), driven by the live Hermes Agent
session plus recent model experimentation

## TL;DR

You're right, and it's worse than "elastically occupying." Over the last 3 hours the
daemon has been in an active **eviction/reload thrashing loop**: `nemotron-3.5-lightning:1m`
(a 33GB model) was loaded and evicted **313 times**, `nomic-embed-text` **256 times**,
and an unexplained `nemotron-3-nano:4b-bf16` **184 times** — largely every 30-90 seconds,
per `journalctl -u ollama`. This isn't nemotron being "sticky"; it's the scheduler
constantly fighting to fit more models than it's allowed to hold at once.

## Root cause

`OLLAMA_MAX_LOADED_MODELS=2` — set (redundantly, see below) in
`/etc/systemd/system/ollama.service.d/local-only.conf` and `override.conf` — caps the
daemon at **2 concurrent resident models**, with `OLLAMA_KEEP_ALIVE=5m`.

But a single Hermes Agent turn with memory enabled can require **up to 3 distinct
models** at once:
1. The active **chat model** (`model.default` — currently `dler-r1-7b-64k`, was
   `dler-r1-7b`, `hermes3:8b-llama3.1-q5_K_M`, `qwen3:4b-thinking-2507-q8_0` at various
   points during testing today).
2. **mem0's extraction LLM**, hardcoded in `mem0.json` (`oss.llm.config.model`) to
   `nemotron-3.5-lightning:1m` — independent of whatever the chat model is, confirmed for
   both the `default` and `memory-lab` profiles.
3. **mem0's embedder**, hardcoded to `nomic-embed-text`.

With a 2-model cap and a 3-model workload, every turn that touches memory forces an
eviction of whichever of the three was loaded longest ago — then the next turn needs it
back, evicting something else. The `journalctl` log is explicit about this:
`"max runners achieved, unloading one to make room"` and
`"resetting model to expire immediately to make room"` appear on essentially every load
cycle in the 3-hour window, not occasionally.

Reloading `nemotron-3.5-lightning:1m` is especially expensive to do this often — it's the
single largest model in the rotation (33GB, 1,048,576-token context slots), so each of
those 313 reloads is a real disk/GPU-transfer cost, not a cheap swap.

## Compounding factor: the "secret sauce" masks it as slowness, not an error

`override.conf` sets `OLLAMA_MAX_VRAM=0`, commented as "don't be conservative, offload
extra layers to system RAM/CPU instead of refusing." This is why nothing ever *errors* —
you just silently get partial CPU offload (the 48%/52% CPU/GPU splits seen consistently in
`ollama ps` this session) instead of an OOM or "won't fit" refusal. That's a reasonable
setting on its own, but combined with the thrashing above, it hides a scheduling problem
behind what looks like "the GPU is just a bit busy" rather than "the daemon is reloading a
33GB model roughly every half-minute."

## Config hygiene issue found along the way

`/etc/systemd/system/ollama.service.d/` has **two separate, overlapping drop-ins**
(`local-only.conf` from 2026-01-24 and `override.conf` from 2026-01-28) both setting
`OLLAMA_MAX_LOADED_MODELS`, `OLLAMA_HOST`, `CUDA_VISIBLE_DEVICES`, `OLLAMA_FLASH_ATTENTION`,
etc. — plus `.backup` copies of older versions of each. They happen to agree on the values
that matter here, but systemd resolves conflicts by file sort order silently, so this is
fragile: a future edit to only one of the two will produce confusing, hard-to-diagnose
behavior. Worth consolidating into one file.

## Open question — not yet explained

`nemotron-3-nano:4b-bf16` cycled 184 times in the same window, but it doesn't appear as a
configured role in `config.yaml`, either profile's `mem0.json`, or anywhere else searched.
It was also pulled ~22 minutes before this investigation (per `ollama list` timestamp),
same window as other model pulls from earlier tasks today. Possible explanations not yet
verified: a `hermes model` interactive benchmarking/selection step, an auto-eval feature,
or leftover manual `ollama run` testing in another shell. Flagging rather than guessing —
worth checking shell history / other terminal sessions if you want this run down.

## Recommendations

1. **Raise `OLLAMA_MAX_LOADED_MODELS` to 3** so chat model + mem0 LLM + embedder can
   coexist without constant eviction — this directly targets the measured cause. Check
   combined VRAM budget first (two GPUs, currently modest free headroom); with
   `OLLAMA_MAX_VRAM=0` already permissive, worst case is more CPU offload, not a crash.
2. **Or: make mem0's extraction LLM cheap instead of huge.** It's currently the single
   most expensive model to keep cycling (33GB). Pointing `mem0.json`'s `oss.llm` at
   something small (e.g. `qwen3:4b-instruct-2507-q8_0`, already pulled) would make each
   eviction/reload of the memory-extraction step nearly free, even if the 2-model cap
   stays as-is. This was previously avoided specifically to skip running "a second heavy
   resident model" for mem0 — but nemotron reloading 313 times is a worse outcome than a
   small model resident full-time.
3. **Consolidate `local-only.conf` and `override.conf`** into one drop-in to remove the
   silent-override fragility, and delete the stale `.backup` files once confirmed unneeded.
4. **Track down `nemotron-3-nano:4b-bf16`'s 184 loads** before concluding the fix in (1)/(2)
   is sufficient — if it's a real, recurring role, it adds to the concurrency math above
   (making the cap-vs-workload mismatch even worse than 2-vs-3).
5. Re-run this same `journalctl` eviction-count check after applying (1) or (2) to confirm
   the thrashing actually stops, rather than assuming the fix worked.

## Evidence commands used

```
ollama ps
journalctl -u ollama --since "-3 hours" --no-pager | grep -E 'msg="loaded runners"|expiring to unload|make room'
journalctl -u ollama --since "-3 hours" --no-pager | grep -oE "runner.name=registry.ollama.ai/library/[^ ]+" | sort | uniq -c | sort -rn
systemctl show ollama -p Environment
cat /etc/systemd/system/ollama.service.d/*.conf
```

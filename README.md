<img src="logo.svg" width="640" alt="vram-thrash-diaries logo">

<p align="center">
  <a href="LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-blue.svg"></a>
  <img alt="Platform" src="https://img.shields.io/badge/platform-Linux-informational">
  <img alt="Local-first AI" src="https://img.shields.io/badge/local--first-AI-8b5cf6">
  <img alt="Made with Ollama" src="https://img.shields.io/badge/made%20with-Ollama-000000">
  <img alt="Diagrams" src="https://img.shields.io/badge/diagrams-30%20%C3%97%202%20formats-orange">
  <img alt="Rendered with Graphviz" src="https://img.shields.io/badge/rendered%20with-Graphviz-2e8b57">
  <a href="https://github.com/danindiana/vram-thrash-diaries/commits/master"><img alt="Last commit" src="https://img.shields.io/github/last-commit/danindiana/vram-thrash-diaries"></a>
  <a href="https://github.com/danindiana/vram-thrash-diaries/issues"><img alt="Issues" src="https://img.shields.io/github/issues/danindiana/vram-thrash-diaries"></a>
  <img alt="Status" src="https://img.shields.io/badge/status-documented-brightgreen">
</p>

# vram-thrash-diaries

A field journal from one long, real debugging session on a dual-GPU local-LLM
box running [Hermes Agent](https://github.com/NousResearch/hermes-agent) on
top of [Ollama](https://ollama.com), with [mem0](https://mem0.ai) for
persistent memory. It started as "why can't I load a 32K-context model" and
ended as a full teardown of *why local model schedulers thrash*, how to
actually diagnose it, and what it costs to fix.

If you run more than one local model behind Ollama — a chat model, an
embedder, a memory-extraction step, maybe a coding agent — and you've ever
looked at `nvidia-smi` and thought *"wait, why does the GPU look idle but
nothing's fast,"* this repo is for you.

## TL;DR

- A 33GB memory-extraction model was silently evicting the active chat model
  **313 times in 3 hours** — on every single memory write, regardless of
  which chat model was active.
- The fix wasn't "buy more VRAM." It was: understand exactly how many
  concurrent model *roles* your workload actually needs, give the cheap ones
  (embedder, memory extraction) a cheap home (CPU/RAM, not GPU), and size
  your scheduler's concurrency cap to match — not bigger, not smaller.
- Along the way: a completely unnoticed second Ollama daemon had been running
  for 6+ hours, silently capable of double-booking a GPU the "real" daemon
  also used. Nobody knew it existed until `ss -tlnp` said otherwise.

## The story, in diagrams

Every diagram exists in **two forms**: a Mermaid version inline in each
`diagrams/*.md` file (renders natively in GitHub), and a **dark-background,
neon-color Graphviz render** (both `.svg` and `.png`) embedded at the top of
that same file — same content, drawn twice, different renderer. Sources for
the latter live in [`diagrams/graphviz/`](diagrams/graphviz).

| # | Diagram | What it shows |
|---|---|---|
| 01 | [System architecture](diagrams/01-system-architecture.md) | The full stack: Hermes Agent + Qwen Code → Ollama → 2 GPUs + mem0/Qdrant |
| 02 | [What we did (timeline)](diagrams/02-what-we-did-timeline.md) | The whole session as a sequence diagram, cause → effect → next cause |
| 03 | [Lessons learned](diagrams/03-lessons-learned.md) | The generalizable takeaways, mindmapped |
| 04 | [GPU scheduling: before/after](diagrams/04-gpu-scheduling-before-after.md) | The measured numbers — 313 evictions → 0, 45GB/52%CPU → 26GB/5%CPU |
| 05 | [mem0 memory pipeline](diagrams/05-mem0-memory-pipeline.md) | How a memory write actually flows, post-fix |
| 06 | [How-to: reproduce](diagrams/06-howto-reproduce.md) | A generic flowchart + command list for diagnosing this on *your* box |
| 07 | [Catch-22s](diagrams/07-catch22s.md) | Every fix here created a narrower new problem before it actually settled |
| 08 | [Tool-calling bridge fix](diagrams/08-tool-calling-bridge-fix.md) | Why small models kept mangling Hermes's tool-call batching layer |
| 09 | [Future directions](diagrams/09-future-directions.md) | What's deliberately left open |
| 10 | [Model-choice decision tree](diagrams/10-model-choice-decision-tree.md) | Every chat-model swap this session made, and why |

### Gallery (dark/neon Graphviz renders)

<table>
<tr>
<td width="50%"><a href="diagrams/01-system-architecture.md"><img src="diagrams/graphviz/png/01-system-architecture.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/02-what-we-did-timeline.md"><img src="diagrams/graphviz/png/02-what-we-did-timeline.png" width="100%"></a></td>
</tr>
<tr>
<td width="50%"><a href="diagrams/03-lessons-learned.md"><img src="diagrams/graphviz/png/03-lessons-learned.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/04-gpu-scheduling-before-after.md"><img src="diagrams/graphviz/png/04-gpu-scheduling-before-after.png" width="100%"></a></td>
</tr>
<tr>
<td width="50%"><a href="diagrams/05-mem0-memory-pipeline.md"><img src="diagrams/graphviz/png/05-mem0-memory-pipeline.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/06-howto-reproduce.md"><img src="diagrams/graphviz/png/06-howto-reproduce.png" width="100%"></a></td>
</tr>
<tr>
<td width="50%"><a href="diagrams/07-catch22s.md"><img src="diagrams/graphviz/png/07-catch22s.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/08-tool-calling-bridge-fix.md"><img src="diagrams/graphviz/png/08-tool-calling-bridge-fix.png" width="100%"></a></td>
</tr>
<tr>
<td width="50%"><a href="diagrams/09-future-directions.md"><img src="diagrams/graphviz/png/09-future-directions.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/10-model-choice-decision-tree.md"><img src="diagrams/graphviz/png/10-model-choice-decision-tree.png" width="100%"></a></td>
</tr>
</table>

## Appendix: adjacent topics

The 10 diagrams above document *this specific session*. These 10 are
general local-LLM-infra concepts that session touched on or implied but
never explained on their own terms — useful even if you never hit the
exact bugs above.

| # | Diagram | What it shows |
|---|---|---|
| 11 | [Quantization tradeoffs](diagrams/appendix/11-quantization-tradeoffs.md) | FP16 → Q8 → Q4 → lower: what each step actually costs |
| 12 | [GPU memory hierarchy](diagrams/appendix/12-gpu-memory-hierarchy.md) | VRAM / PCIe / system RAM / disk, and why the CPU/GPU split happens |
| 13 | [Ollama scheduler internals](diagrams/appendix/13-ollama-scheduler-internals.md) | The actual per-layer fit-testing loop behind `common_params_fit_impl` |
| 14 | [Agent harness comparison](diagrams/appendix/14-agent-harness-comparison.md) | Hermes Agent vs. Qwen Code, architecturally |
| 15 | [MoE vs. dense](diagrams/appendix/15-moe-vs-dense.md) | Why a 30B MoE model still needs 30B-worth of VRAM |
| 16 | [mem0 architecture](diagrams/appendix/16-mem0-architecture.md) | OSS (self-hosted) vs. platform (hosted) mode, and why OSS fit here |
| 17 | [systemd hardening patterns](diagrams/appendix/17-systemd-hardening-patterns.md) | The overlapping-drop-ins anti-pattern, generalized |
| 18 | [LAN exposure security](diagrams/appendix/18-lan-exposure-security.md) | What `OLLAMA_HOST=0.0.0.0` actually exposes, and to whom |
| 19 | [Context window / KV-cache math](diagrams/appendix/19-context-window-kv-cache-math.md) | The formula behind the 45GB→26GB win |
| 20 | [Tool-calling protocol comparison](diagrams/appendix/20-tool-calling-protocol-comparison.md) | OpenAI / Hermes ChatML / Anthropic tool-use, side by side |

### Appendix gallery

<table>
<tr>
<td width="50%"><a href="diagrams/appendix/11-quantization-tradeoffs.md"><img src="diagrams/appendix/graphviz/png/11-quantization-tradeoffs.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/appendix/12-gpu-memory-hierarchy.md"><img src="diagrams/appendix/graphviz/png/12-gpu-memory-hierarchy.png" width="100%"></a></td>
</tr>
<tr>
<td width="50%"><a href="diagrams/appendix/13-ollama-scheduler-internals.md"><img src="diagrams/appendix/graphviz/png/13-ollama-scheduler-internals.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/appendix/14-agent-harness-comparison.md"><img src="diagrams/appendix/graphviz/png/14-agent-harness-comparison.png" width="100%"></a></td>
</tr>
<tr>
<td width="50%"><a href="diagrams/appendix/15-moe-vs-dense.md"><img src="diagrams/appendix/graphviz/png/15-moe-vs-dense.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/appendix/16-mem0-architecture.md"><img src="diagrams/appendix/graphviz/png/16-mem0-architecture.png" width="100%"></a></td>
</tr>
<tr>
<td width="50%"><a href="diagrams/appendix/17-systemd-hardening-patterns.md"><img src="diagrams/appendix/graphviz/png/17-systemd-hardening-patterns.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/appendix/18-lan-exposure-security.md"><img src="diagrams/appendix/graphviz/png/18-lan-exposure-security.png" width="100%"></a></td>
</tr>
<tr>
<td width="50%"><a href="diagrams/appendix/19-context-window-kv-cache-math.md"><img src="diagrams/appendix/graphviz/png/19-context-window-kv-cache-math.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/appendix/20-tool-calling-protocol-comparison.md"><img src="diagrams/appendix/graphviz/png/20-tool-calling-protocol-comparison.png" width="100%"></a></td>
</tr>
</table>

## Appendix II: still more adjacent topics

A third batch — general local-LLM concepts referenced throughout the repo
(embeddings, Modelfiles, GGUF, thinking models, multi-GPU strategies,
prompt caching, ...) that hadn't gotten their own diagram yet.

| # | Diagram | What it shows |
|---|---|---|
| 21 | [Embeddings & vector search](diagrams/appendix/21-embeddings-vector-search.md) | How `nomic-embed-text` + Qdrant actually turn text into searchable memory |
| 22 | [Model lifecycle & keep-alive](diagrams/appendix/22-model-lifecycle-keepalive.md) | The state machine behind every `journalctl` load/evict line in this repo |
| 23 | [Modelfile anatomy](diagrams/appendix/23-modelfile-anatomy.md) | The 5 directives behind every derived tag this session created |
| 24 | [GGUF file anatomy](diagrams/appendix/24-gguf-file-anatomy.md) | What `ollama show` is actually reading out of the file on disk |
| 25 | [Thinking models explained](diagrams/appendix/25-thinking-models-explained.md) | What "-thinking" means, and the hard error one model hit for lacking it |
| 26 | [Multi-GPU parallelism strategies](diagrams/appendix/26-multi-gpu-parallelism-strategies.md) | Tensor-split (used here) vs. pipeline vs. the stray daemon's accidental data-parallel |
| 27 | [Observability practices](diagrams/appendix/27-observability-practices.md) | The three log layers that found every bug in this repo |
| 28 | [Model storage & deduplication](diagrams/appendix/28-model-storage-layout.md) | Why deriving new tags this session cost ~0 extra disk |
| 29 | [Local vs. cloud decision framework](diagrams/appendix/29-local-vs-cloud-decision.md) | An honest framework for when local *isn't* the right call |
| 30 | [Prompt/context caching](diagrams/appendix/30-prompt-context-caching.md) | Why every eviction was more costly than the reload time alone suggested |

### Appendix II gallery

<table>
<tr>
<td width="50%"><a href="diagrams/appendix/21-embeddings-vector-search.md"><img src="diagrams/appendix/graphviz/png/21-embeddings-vector-search.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/appendix/22-model-lifecycle-keepalive.md"><img src="diagrams/appendix/graphviz/png/22-model-lifecycle-keepalive.png" width="100%"></a></td>
</tr>
<tr>
<td width="50%"><a href="diagrams/appendix/23-modelfile-anatomy.md"><img src="diagrams/appendix/graphviz/png/23-modelfile-anatomy.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/appendix/24-gguf-file-anatomy.md"><img src="diagrams/appendix/graphviz/png/24-gguf-file-anatomy.png" width="100%"></a></td>
</tr>
<tr>
<td width="50%"><a href="diagrams/appendix/25-thinking-models-explained.md"><img src="diagrams/appendix/graphviz/png/25-thinking-models-explained.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/appendix/26-multi-gpu-parallelism-strategies.md"><img src="diagrams/appendix/graphviz/png/26-multi-gpu-parallelism-strategies.png" width="100%"></a></td>
</tr>
<tr>
<td width="50%"><a href="diagrams/appendix/27-observability-practices.md"><img src="diagrams/appendix/graphviz/png/27-observability-practices.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/appendix/28-model-storage-layout.md"><img src="diagrams/appendix/graphviz/png/28-model-storage-layout.png" width="100%"></a></td>
</tr>
<tr>
<td width="50%"><a href="diagrams/appendix/29-local-vs-cloud-decision.md"><img src="diagrams/appendix/graphviz/png/29-local-vs-cloud-decision.png" width="100%"></a></td>
<td width="50%"><a href="diagrams/appendix/30-prompt-context-caching.md"><img src="diagrams/appendix/graphviz/png/30-prompt-context-caching.png" width="100%"></a></td>
</tr>
</table>

## The measured wins

| Metric | Before | After |
|---|---|---|
| nemotron model evictions (3h window) | 313 | 0 |
| Embedder evictions (3h window) | 256 | 0 |
| A 4B "thinking" model's reported footprint | 45 GB | 26 GB |
| That same model's CPU/GPU compute split | 48% / 52% | **5% / 95%** |
| Concurrent model roles supported without eviction | 1-2 (fighting) | 3 (coexisting) |

None of this required new hardware. It required actually reading
`journalctl -u ollama`, correlating it against `mem0`'s own logs, and
questioning assumptions ("the docs say per-GPU" → tested it → it wasn't,
in practice, for a CPU-only runner).

## Catch-22s worth knowing about before you hit them yourself

- **Capping concurrent models fixes GPU/desktop contention but can starve a
  background service** (memory extraction) that quietly needs its own slot.
- **Making that background service CPU-only doesn't automatically exempt it**
  from a model-count cap that doesn't distinguish devices — you have to
  raise the cap *and* move the work off-GPU together.
- **A documented env var may not be a live one.** One long-standing "secret
  sauce" setting in this stack turned out to be unrecognized by the
  installed Ollama version's own `serve --help` — it had been silently
  doing nothing.

Full writeup: [diagram 07](diagrams/07-catch22s.md).

## How to reproduce this diagnosis on your own machine

Short version — full flowchart and exact commands in
[diagram 06](diagrams/06-howto-reproduce.md):

```bash
ollama ps                                                    # what's loaded right now
journalctl -u ollama --since "-3 hours" --no-pager \
  | grep -E 'loaded runners|make room'                        # is it thrashing?
journalctl -u ollama --since "-3 hours" --no-pager \
  | grep -oE "runner.name=registry.ollama.ai/library/[^ ]+" \
  | sort | uniq -c | sort -rn                                  # which model(s), how often
ss -tlnp | grep -i ollama                                     # more than one daemon?
nvidia-smi -L                                                 # what GPUs actually exist
```

## What's in this repo

- [`SESSION.md`](SESSION.md) — the full, unabridged session log.
- [`ollama-model-thrashing-nemotron.md`](ollama-model-thrashing-nemotron.md) —
  the original diagnostic report on the eviction thrashing.
- [`hermes-default-model-ornith-vs-hermes3.md`](hermes-default-model-ornith-vs-hermes3.md) —
  a research writeup on chat-model choice (`hermes3:8b` vs. `ornith-1.5:9b`),
  including a real-world benchmark-vs-reliability gap that's easy to miss if
  you only read leaderboards.
- [`diagrams/`](diagrams) — the 10 diagrams above, each a standalone
  Markdown file with a Mermaid diagram GitHub renders natively.

## What's still open

- The actual long-term default chat model is undecided — see
  [diagram 10](diagrams/10-model-choice-decision-tree.md).
- [Qwen Code](https://github.com/QwenLM/qwen-code) got installed and wired to
  the same local Ollama daemon, confirmed working, but hasn't been compared
  head-to-head against Hermes Agent yet.
- A stray, forgotten systemd unit (a second Ollama daemon, 6.5 hours old,
  silently capable of double-booking a GPU) was found and disabled — a good
  reminder to periodically audit `systemctl list-units | grep -i ollama` on
  any box that's had a few months of "quick, let's try this" changes.

---

*Written up from a real session — nothing here is hypothetical or a
benchmark synthetic. Hostname genericized; technical values, configs, and
numbers are real.*

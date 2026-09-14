# Hermes Agent default-profile chat model: hermes3:8b vs. ornith-1.5:9b

Date: 2026-09-13
Scope: `default` Hermes Agent profile on the-box (`~/.hermes/config.yaml`)

## 1. Current state

- `default` profile's `model.default` is currently `dler-r1-7b-64k:latest` (session-local
  Ollama tag, `num_ctx=65536`, set 2026-09-13 to fix a context-window error — not a
  considered choice for daily driving, just what unblocked the immediate ask).
- A prior, more deliberate recommendation (session_1789339399, **not yet applied**) was to
  switch the default to `hermes3:8b-llama3.1-q8_0`, based on `hermes insights` showing the
  real workload is 56.6% `terminal` tool calls plus execute_code/read_file/write_file/
  web_search — i.e. a tool-calling agent workload, not deep reasoning — and the prior
  defaults (`nemotron-3.5-lightning:1m` / `muse-glimmer:30b`) are 30B-class, forcing a
  dual-GPU tensor split for no measured benefit on that workload.

## 2. Why hermes3:8b was the pick, and why it's showing its age

`hermes3:8b-llama3.1-q8_0` (already pulled, 8.5GB, single-GPU fit) is Nous Research's own
fine-tune, in the same tool-call format Hermes Agent itself expects — the rationale was
compatibility + fit, not raw capability.

It's built on **Llama 3.1 8B**, a 2024-era base. Ollama's local mtime for this tag reads
"10 months ago," consistent with it predating this evaluation by most of a year. In a
fast-moving small-model landscape, an 8B-class model this old is a reasonable thing to
second-guess — hence this doc.

## 3. Ornith-1.5-9B — what it is

- Dense ~9B model from ornith-ai, built on **Qwen3.5 + Gemma4** foundations (continued
  pretraining/mid-training/post-training), MIT licensed.
- Trained via a **self-improving RL loop**: the model generates its own tasks/scaffolds/
  solutions rather than a fixed human-curated training harness — same approach across the
  Ornith 1.5 family (a larger MoE variant also exists).
- Context: 262,144 tokens native; YaRN scaling (factor 4.0) extends to ~1M.
- Tool-calling: fully OpenAI-compatible `tool_calls`, via `qwen3_xml`/`qwen3_coder` parsers
  in vLLM/SGLang. Ollama pull: `ollama run ornith-1.5:9b` (official, syncs
  `ornith-ai/Ornith-1.5-9B-GGUF`). GGUF quant sizes run ~5.9GB (Q4_K_M) up to ~9-10GB
  (Q6_K/Q8), comparable footprint to `hermes3:8b-llama3.1-q8_0`.
- **Multiple unrelated/unofficial HF repos also use the "Ornith 1.5" name** — if pulling
  manually rather than via `ollama run ornith-1.5:9b`, verify the publisher is `ornith-ai`
  or a recognized quantizer (bartowski, etc.), not an impersonator.

### Published benchmarks

| Benchmark | Score |
|---|---|
| SWE-bench Verified | 70.6 |
| SWE-bench Pro | 47.5 |
| Terminal-Bench 2.1 (Terminus-2) | 46.2 |
| GPQA Diamond | 86.4 |
| MCP-Atlas (agentic) | 54.2 |
| ClawEval (coding) | 66.5 |

On paper this beats similarly-sized models (Qwen3.5-9B, Gemma-4-31B) and closes in on
Qwen3.6-35B-A3B on several agentic/coding benchmarks — a much stronger sheet than
`hermes3:8b` (Llama 3.1-class) would post today.

## 4. The catch: a documented benchmark-vs-reality gap

An independent local test (MindStudio, Aug 2026) ran Ornith-1.5-9B on two open-ended
agentic tasks — provisioning a free-tier AWS EC2 instance via CLI, and generating a
self-contained HTML animation:

- **AWS task**: couldn't identify the right AMI from public options, made repeated command
  typos, burned ~43,000 tokens over 7 minutes, then emitted "hallucinated text in Chinese
  rather than a valid command." Looped between tools even after being manually given the
  correct AMI. Never completed.
- **Coding task**: produced "garbled generation and low-quality results" on a simple HTML
  animation — notably worse than Ornith 1.5's own larger MoE sibling on the same prompt.

Their stated root cause: published benchmarks test narrower, more structured scenarios than
open-ended real work requiring ambiguity handling, large unstructured context, and
multi-step error recovery — exactly the shape of Hermes Agent's actual measured workload
(terminal/exec/read/write tool calls, 56.6%+ of calls). This is the same axis the
benchmark gap was measured on, not a tangential concern.

(Their test ran full-precision-ish safetensors on an A100 via vLLM, not a quantized GGUF
on Ollama — so quantization could make local results better or worse than what they saw;
it's not a controlled comparison to our setup.)

## 5. Recommendation

Don't swap the default straight to `ornith-1.5:9b` on the strength of its benchmark sheet
alone — the one hands-on report available shows it failing on the exact task shape (open-
ended, multi-step, tool-heavy) that dominates this Hermes Agent's actual usage, despite
strong leaderboard numbers on paper. That's a specific, relevant red flag, not generic
skepticism of benchmarks.

Suggested path:
1. Keep `hermes3:8b-llama3.1-q8_0` as the safe default for now — it's old but it's a known
   quantity already validated for this workload.
2. Pull `ornith-1.5:9b` (official tag) and trial it in a sandboxed profile (a new profile,
   not `memory-lab` — that one's scoped to memory-provider experiments) against a handful
   of real `terminal`/`execute_code` tasks from recent `hermes insights` logs, before
   touching the default.
3. If it holds up on those, promote it; if not, look at `qwen3.5:9b` or similar as a more
   conservative "newer than hermes3:8b" step — same landscape search flagged it as the
   safe/solid pick in the 8-9B tool-calling range without Ornith's self-improving-RL novelty
   (and its attendant reliability question mark).
4. Either way, revert `model.default` off the current `dler-r1-7b-64k:latest` — that tag
   only exists to fix a context-window bug from an unrelated session and was never
   evaluated as a chat-quality choice.

## Sources

- [Ornith 1.5 9B: Local Test Results Expose a Benchmark Gap (MindStudio)](https://www.mindstudio.ai/blog/ornith-1-5-9b-local-test)
- [ornith-ai/Ornith-1.5-9B (Hugging Face)](https://huggingface.co/ornith-ai/Ornith-1.5-9B)
- [ornith-ai/Ornith-1.5-9B-GGUF (Hugging Face)](https://huggingface.co/ornith-ai/Ornith-1.5-9B-GGUF)
- [How to Run Ornith 1.5 9B Locally (Atomic Chat)](https://atomic.chat/blog/guides/how-to-run-ornith-1-5-locally)
- [Best Local Models for Tool Calling in 2026 (PromptQuorum)](https://www.promptquorum.com/power-local-llm/best-local-models-tool-calling-2026)

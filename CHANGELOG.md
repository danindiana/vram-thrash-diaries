# Changelog

All notable changes to this repo are documented here. Format loosely follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

- Added `.gitattributes` (linguist stats, DOT file attribution, generated-image
  markers), `CITATION.cff`, `.gitignore`, `.editorconfig`.
- Added `.github/workflows/verify-diagrams.yml` — CI that re-renders every `.dot`
  source and fails if the SVG output no longer matches the committed file (plus a
  render-smoke-test for PNG).
- Added `.github/ISSUE_TEMPLATE/` (diagram error, related-topic suggestion) and
  `.github/PULL_REQUEST_TEMPLATE.md`.
- Added a generated social-preview image (`.github/social-preview.png`).
- Added this changelog.

## 2026-09-14

- README: added GPU/VRAM hardware badges, repo-size and stars badges, fixed the
  stale "30 diagrams" badge to the real count (50), added a table of contents and a
  `hermes-goal-judge/` section.
- Added `hermes-goal-judge/` — a second writeup thread: `01-judge-investigation/`
  (why Hermes Agent's `/goal` loop uses a separate auxiliary "judge" LLM call) and
  `02-outage-fix/` (a real `/goal` outage diagnosis and fix, plus a caught
  context-window regression), 20 diagrams total.

## 2026-09-13

- Added appendix diagrams 21-30: embeddings/vector search, model lifecycle,
  Modelfile anatomy, GGUF anatomy, thinking-model mechanics, multi-GPU parallelism,
  observability, model storage, local-vs-cloud framework, prompt caching.
- Added appendix diagrams 11-20: quantization tradeoffs, GPU memory hierarchy,
  Ollama scheduler internals, agent harness comparison, MoE vs. dense, mem0
  architecture, systemd hardening, LAN exposure security, KV-cache math,
  tool-calling protocol comparison.
- Added dark/neon Graphviz renders (SVG + PNG) for the original 10 diagrams.
- Initial writeup: diagnosing and fixing an Ollama model-loading eviction-thrashing
  bug on a dual-GPU Hermes Agent box (313 evictions/3h → 0), plus the original 10
  Mermaid diagrams, README, logo, LICENSE.

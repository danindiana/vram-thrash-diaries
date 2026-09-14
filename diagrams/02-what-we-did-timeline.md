# 02 — What We Did (Timeline)

<img src="graphviz/png/02-what-we-did-timeline.png" alt="02 — What We Did (Timeline) (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/02-what-we-did-timeline.svg) · [PNG](graphviz/png/02-what-we-did-timeline.png) · [DOT source](graphviz/02-what-we-did-timeline.dot)*

One session, one long chain of "fix this → notice that → fix that too."
Every arrow below is a real cause-and-effect link, not just chronology.

```mermaid
sequenceDiagram
    participant U as Operator
    participant H as Hermes Agent
    participant O as Ollama daemon

    U->>H: Switch model to dler-r1-7b
    H-->>U: Error: 32K context < 64K minimum
    U->>O: Build dler-r1-7b-64k (num_ctx=65536)
    Note over U,O: Context bug fixed — not a real model choice

    U->>O: "nemotron seems to elastically occupy resources?"
    U->>O: journalctl investigation
    O-->>U: 313 nemotron evictions / 3h (MAX_LOADED_MODELS=2 vs 3-model workload)
    U->>O: Consolidate systemd drop-ins, cap=1, reserve 1GiB/GPU for desktop
    Note over O: Thrashing stops — but new tradeoff introduced

    U->>O: "cut qwen3:4b's context to fit more in VRAM?"
    U->>O: NUM_PARALLEL 2→1 instead
    O-->>U: 45GB→26GB, CPU/GPU split 48/52%→5/95%
    Note over O: Real, measured win

    U->>O: "nemotron loads on every turn, right?"
    O-->>U: Confirmed — mem0 extraction call, hardcoded to nemotron
    U->>O: Build mem0-extractor-cpu (num_gpu=0), rewire mem0.json
    Note over O: But MAX_LOADED_MODELS=1 evicts it too (CPU or not)
    U->>O: Raise cap to 3
    O-->>U: Chat model + embedder + CPU extractor coexist, zero eviction

    U->>O: Diagnose live tool-call errors
    O-->>U: qwen3:4b-thinking malforms tool_call bridge, repeatedly
    U->>H: Disable tools.tool_search (bridge removed entirely)

    U->>O: Investigate "GPU idle, model not on it"
    O-->>U: Found stray ollama-gpu1.service (6.5h old, wrong GPU description)
    U->>O: Stop + disable it

    U->>H: Install & wire Qwen Code to local Ollama
    H-->>U: Confirmed working end-to-end (tool calls, local model)
```

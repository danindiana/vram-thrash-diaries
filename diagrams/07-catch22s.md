# 07 — Catch-22s

<img src="graphviz/png/07-catch22s.png" alt="07 — Catch-22s (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/07-catch22s.svg) · [PNG](graphviz/png/07-catch22s.png) · [DOT source](graphviz/07-catch22s.dot)*

Every fix in this session created a new, narrower problem before things
actually settled. This is the loop.

```mermaid
flowchart TD
    P1["Problem: MAX_LOADED_MODELS=2\nvs 3-role workload → thrashing"] --> F1["Fix: cap to 1,\nreserve GPU headroom for desktop"]
    F1 --> P2["New problem: cap=1 means\nchat model and mem0's LLM\ncan NEVER coexist —\nevery memory call evicts the chat model"]
    P2 --> F2["Fix: make mem0's LLM\nCPU-only (0 VRAM)"]
    F2 --> P3["New problem: the cap counts\nrunners by COUNT, not by device —\nCPU-only still evicts / gets evicted\nto stay under cap=1"]
    P3 --> F3["Fix: raise cap to 3\n(now cheap enough since 2 of the 3\nroles are small/CPU)"]
    F3 --> R["Resolved: chat + embedder + CPU extractor\ncoexist, 0 eviction"]

    style P2 fill:#fff3e0,stroke:#e65100
    style P3 fill:#fff3e0,stroke:#e65100
    style R fill:#e8f5e9,stroke:#2e7d32
```

**The general shape of the trap:** a resource cap meant to solve contention
between the "expensive, important" tenant (chat model) and an "external"
concern (desktop GPU use) doesn't know about a *third* tenant (memory
extraction) until it collides with it — and the naive fix for that
collision (make the third tenant cheap) doesn't actually escape the cap
unless the cap itself understands *why* something is cheap.

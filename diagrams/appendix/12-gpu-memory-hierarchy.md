# 12 — GPU Memory Hierarchy

<img src="graphviz/png/12-gpu-memory-hierarchy.png" alt="12 — GPU Memory Hierarchy (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/12-gpu-memory-hierarchy.svg) · [PNG](graphviz/png/12-gpu-memory-hierarchy.png) · [DOT source](graphviz/12-gpu-memory-hierarchy.dot)*

The "CPU/GPU split" percentage you see in `ollama ps` is really a statement
about how many layers crossed which boundary below. This is the general
picture; [diagram 04](../04-gpu-scheduling-before-after.md) has this
session's actual before/after numbers.

```mermaid
flowchart TD
    L1["GPU VRAM (on-die/on-package)\nfastest, smallest\n12-16GB this box"]
    L2["PCIe bus\nGPU ↔ CPU transfer\nGen4 x16 ≈ 32GB/s"]
    L3["System RAM\nslower, huge\n176GB free this box"]
    L4["NVMe / disk\nmodel weights at rest\nmmap'd on load"]

    L1 -->|"offloaded layers cross\nthis boundary (the\nCPU/GPU % split)"| L2 --> L3 -->|cold load| L4

    subgraph impl["What this meant in practice"]
        I1["'48%/52% CPU/GPU split' = half\nthe layers crossed L1→L2\nevery forward pass — slow"]
        I2["Fixing NUM_PARALLEL cut KV-cache\nsize, so more fit in L1 alone →\n95% GPU, minimal L2 crossing"]
    end
```

**Why this matters practically:** every layer that doesn't fit in VRAM has
to shuttle activations across PCIe on every forward pass. That's not a
fixed one-time cost — it's paid per token generated, which is exactly why
the CPU/GPU split percentage correlates so directly with perceived speed.

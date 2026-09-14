# 15 — MoE vs. Dense Model Tradeoffs

<img src="graphviz/png/15-moe-vs-dense.png" alt="15 — MoE vs. Dense Model Tradeoffs (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/15-moe-vs-dense.svg) · [PNG](graphviz/png/15-moe-vs-dense.png) · [DOT source](graphviz/15-moe-vs-dense.dot)*

`qwen3-coder:30b-a3b-q8_0` (used to test Qwen Code) is a Mixture-of-Experts
model — the `-a3b` suffix means ~3B active parameters per token despite
30B total. This diagram covers why that didn't translate into a small VRAM
footprint.

```mermaid
flowchart TD
    subgraph dense["Dense (e.g. qwen3:4b-thinking)"]
        D1["Every parameter active\nfor every token"]
        D2["Smaller total size,\npredictable VRAM/compute"]
        D3["Simpler to place\non a fixed GPU budget"]
        D1 --> D2 --> D3
    end

    subgraph moe["MoE (e.g. qwen3-coder:30b-a3b)"]
        M1["Only a few 'expert' sub-networks\nactivate per token"]
        M2["Large TOTAL param count (30B)\nbut cheaper compute per token"]
        M3["Full weights still must be\nresident (or offloaded) —\nVRAM cost ≈ total size, not active size"]
        M1 --> M2 --> M3
    end

    M3 -.-> OBS["Observed this session: 30b-a3b\nreported 46GB, 45%/55% CPU/GPU —\nMoE's compute-per-token win doesn't\nshrink the VRAM footprint"]
```

**The gotcha in one sentence:** MoE makes *inference compute* cheaper per
token, not *memory footprint* smaller — all experts have to be loadable
even though only a few fire per token, so don't reach for an MoE tag
expecting it to be VRAM-friendly just because its active-parameter count
sounds small.

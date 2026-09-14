# 11 — Quantization Formats & Tradeoffs

<img src="graphviz/png/11-quantization-tradeoffs.png" alt="11 — Quantization Formats & Tradeoffs (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/11-quantization-tradeoffs.svg) · [PNG](graphviz/png/11-quantization-tradeoffs.png) · [DOT source](graphviz/11-quantization-tradeoffs.dot)*

Every model tag mentioned in the main writeup (`q8_0`, `q5_K_M`, `q4_K_M`, ...)
is a point on this spectrum. This diagram is a general reference for what
those suffixes actually trade off — it's not specific to any one model here.

```mermaid
flowchart LR
    FP16["FP16 / BF16\n~2 bytes/param\nfull reference quality"]
    Q8["Q8_0\n~1 byte/param\nnear-lossless, ~2x smaller"]
    Q6["Q6_K\n~0.75 byte/param\nvery close to Q8"]
    Q5["Q5_K_M\n~0.65 byte/param\ngood balance"]
    Q4["Q4_K_M\n~0.55 byte/param\nnoticeable but usable loss"]
    IQ4["IQ4_XS / lower\n~0.5 byte/param\naggressive, quality risk rises"]

    FP16 -->|smaller / faster,\nmore quality risk| Q8 --> Q6 --> Q5 --> Q4 --> IQ4

    subgraph used["This session used"]
        N1["Q8_0 for most chat models\n(dler-r1-7b, qwen3:4b-thinking)"]
        N2["Q5_K_M for one hermes3:8b\nvariant (failed on 'thinking' param —\nnot a quantization issue)"]
    end
```

**The rule of thumb:** Q8_0 is close enough to full precision that it's a
safe default when VRAM allows. Below Q5, quality loss becomes model- and
task-dependent — worth actually testing on your workload rather than
assuming a smaller number is "fine."

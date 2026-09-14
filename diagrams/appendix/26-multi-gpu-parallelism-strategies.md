# 26 — Multi-GPU Parallelism Strategies

<img src="graphviz/png/26-multi-gpu-parallelism-strategies.png" alt="26 — Multi-GPU Parallelism Strategies (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/26-multi-gpu-parallelism-strategies.svg) · [PNG](graphviz/png/26-multi-gpu-parallelism-strategies.png) · [DOT source](graphviz/26-multi-gpu-parallelism-strategies.dot)*

Three genuinely different ways to spread work across more than one GPU —
this stack uses the first, and this session's stray daemon accidentally
demonstrated a broken version of the third.

```mermaid
flowchart TB
    subgraph tensor["Tensor-split (used this stack)"]
        T1["ONE model, layers divided\nacross GPUs by VRAM fit"]
        T2["Every forward pass touches\nboth GPUs — PCIe traffic\nper token"]
        T3["What OLLAMA_SCHED_SPREAD=true\n+ NUM_GPU=2 does here"]
        T1 --> T2 --> T3
    end

    subgraph pipeline["Pipeline parallel (not used)"]
        P1["Model split into stages,\neach stage on one GPU"]
        P2["Requires careful batching to\nkeep both GPUs busy\n(bubble/stall risk otherwise)"]
        P1 --> P2
    end

    subgraph dataparallel["Data-parallel / one-model-per-GPU\n(the OLD misconfigured setup)"]
        D1["Different model instances,\none per GPU, independent"]
        D2["This session's stray\nollama-gpu1.service accidentally\ndouble-booked GPU1 this way"]
        D1 --> D2
    end
```

**Why tensor-split, specifically, here:** it's the natural fit for "one
model bigger than either GPU alone" — the actual situation this box is in
with 16GB+12GB across two cards. Pipeline parallelism would need explicit
batching logic Ollama doesn't implement for single-request serving, and
data-parallel (independent model copies) only makes sense with genuinely
separate daemons — which is exactly the mistake the stray `ollama-gpu1`
daemon represented, not a real strategy anyone chose on purpose.

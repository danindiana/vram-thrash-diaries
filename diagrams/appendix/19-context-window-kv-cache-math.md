# 19 — Context Window / KV-Cache Math

<img src="graphviz/png/19-context-window-kv-cache-math.png" alt="19 — Context Window / KV-Cache Math (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/19-context-window-kv-cache-math.svg) · [PNG](graphviz/png/19-context-window-kv-cache-math.png) · [DOT source](graphviz/19-context-window-kv-cache-math.dot)*

The formula behind why `OLLAMA_NUM_PARALLEL` turned out to matter more
than `num_ctx` for VRAM in this session's biggest measured win — see
[diagram 04](../04-gpu-scheduling-before-after.md) for the headline numbers.

```mermaid
flowchart TD
    F["KV-cache size ≈\nnum_ctx × num_parallel ×\nlayers × kv_heads × head_dim × 2 (K+V) × dtype_bytes"]

    A["num_ctx: max tokens\nper conversation slot"] --> F
    B["num_parallel: how many\nslots reserved at once\n(concurrent requests)"] --> F
    C["Model architecture:\nlayers × kv_heads × head_dim\n(fixed per model)"] --> F

    subgraph measured["Measured this session (qwen3:4b-thinking)"]
        M1["num_parallel=2, num_ctx=131072:\n45GB reported, 48%/52% CPU/GPU"]
        M2["num_parallel=1, same num_ctx:\n26GB reported, 5%/95% CPU/GPU"]
        M1 -->|"halved parallel slots\n≈ halved KV-cache"| M2
    end

    F -.-> M1
```

**The counterintuitive takeaway:** `num_parallel` is a straight *multiplier*
on KV-cache size — every additional concurrent slot reserves a full extra
copy of the cache, regardless of whether it's ever used. Trimming
`num_ctx` helps too, but linearly and only up to how much of that context
you actually need; trimming an *unused* parallel slot is close to free
performance. Check whether your workload is actually concurrent before
paying for slots you don't need.

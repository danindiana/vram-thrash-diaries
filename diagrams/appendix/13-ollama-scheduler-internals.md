# 13 — Ollama's Layer-Fitting Scheduler

<img src="graphviz/png/13-ollama-scheduler-internals.png" alt="13 — Ollama's Layer-Fitting Scheduler (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/13-ollama-scheduler-internals.svg) · [PNG](graphviz/png/13-ollama-scheduler-internals.png) · [DOT source](graphviz/13-ollama-scheduler-internals.dot)*

This is what's actually happening inside the `common_params_fit_impl` log
lines this session's `journalctl` output was full of — the per-layer test
allocation loop that decides GPU vs. CPU placement for every layer, on
every model load.

```mermaid
flowchart TD
    A["Model load requested"] --> B["common_params_fit_impl:\ntest-allocate memory per device"]
    B --> C["Fill dense layers back-to-front,\none device at a time"]
    C --> D["Per candidate layer count:\nrun a test allocation,\ncheck it fits free VRAM"]
    D --> E{"Layer fits\non this GPU?"}
    E -->|yes| F["Assign layer to GPU,\ntry next layer"]
    E -->|no| G["Stop — remaining layers\nfall back to CPU_Mapped"]
    F --> D
    F --> H["Repeat across all GPUs\n(CUDA0, CUDA1, ...)"]
    G --> H
    H --> I["Load tensors per the\nfinal per-device layer plan"]
```

**Why it matters for capacity planning:** this fit calculation runs *fresh
on every load*, not once. Anything that changes available VRAM between
loads (another model resident, desktop apps, the `OLLAMA_GPU_OVERHEAD`
reservation) changes the outcome — which is exactly why the eviction
thrashing documented in the main writeup caused such wildly inconsistent
CPU/GPU splits load to load.

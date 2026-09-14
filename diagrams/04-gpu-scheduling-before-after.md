# 04 — GPU Scheduling: Before vs. After

Two separate measured comparisons from this session, shown side by side.

```mermaid
flowchart LR
    subgraph Before["BEFORE — thrashing"]
        direction TB
        B1["OLLAMA_MAX_LOADED_MODELS=2\n(2 overlapping systemd drop-ins)"]
        B2["Workload needs 3 concurrent roles:\nchat model + mem0-LLM + embedder"]
        B3["Result: 313 nemotron evictions / 3h\n~every 30-90 seconds"]
        B4["OLLAMA_NUM_PARALLEL=2\nqwen3:4b-thinking: 45GB, 48%/52% CPU/GPU"]
        B1 --> B2 --> B3
    end

    subgraph After["AFTER — stable"]
        direction TB
        A1["Consolidated systemd config\nMAX_LOADED_MODELS=3, GPU_OVERHEAD=1GiB/GPU"]
        A2["mem0 extraction moved to CPU-only model\n(0 VRAM, doesn't compete)"]
        A3["Result: chat + embedder + extractor\nresident simultaneously, 0 evictions"]
        A4["OLLAMA_NUM_PARALLEL=1\nqwen3:4b-thinking: 26GB, 5%/95% CPU/GPU"]
        A1 --> A2 --> A3
    end

    Before -. the fix chain .-> After
```

**The numbers that matter:**

| Metric | Before | After |
|---|---|---|
| nemotron evictions (3h window) | 313 | 0 |
| `nomic-embed-text` evictions (3h) | 256 | 0 |
| qwen3:4b-thinking reported size | 45 GB | 26 GB |
| qwen3:4b-thinking CPU/GPU split | 48% / 52% | 5% / 95% |
| Free VRAM per GPU at idle | ~2GB | ~1-1.6GB (reserved on purpose) |

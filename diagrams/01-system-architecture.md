# 01 — System Architecture

The stack this whole saga lives in: one Ollama daemon serving two GPUs,
Hermes Agent as the interactive driver, and mem0 riding alongside it for
persistent memory. Everything shown here talks to the *same* Ollama daemon
on `:11434` — there is deliberately no second daemon anymore (see
[diagram 07](07-catch22s.md) and the SESSION.md writeup on the stray
`ollama-gpu1.service`).

```mermaid
flowchart TB
    subgraph Client["Interactive clients"]
        HA["Hermes Agent\n(CLI, tool loop, mem0 plugin)"]
        QC["Qwen Code\n(CLI coding agent)"]
    end

    subgraph Daemon["Ollama daemon :11434 (systemd: ollama.service)"]
        SCHED["Scheduler\nOLLAMA_MAX_LOADED_MODELS=3\nOLLAMA_NUM_PARALLEL=1\nOLLAMA_GPU_OVERHEAD=1GiB/GPU"]
    end

    subgraph GPUs["GPUs"]
        GPU0["GPU 0\nRTX 4080 SUPER (16GB)"]
        GPU1["GPU 1\nRTX 5070 (12GB)"]
    end

    CPU["CPU / System RAM\n(176GB free)"]

    subgraph Memory["Memory layer (mem0, OSS mode)"]
        EXTRACT["mem0-extractor-cpu\n(qwen3:4b, num_gpu=0)"]
        EMBED["nomic-embed-text\n(small, GPU)"]
        QDRANT["Qdrant (embedded, on-disk)\nper-profile store"]
    end

    HA -->|OpenAI-compatible API| Daemon
    QC -->|OpenAI-compatible API| Daemon
    SCHED -->|chat model, tensor-split| GPU0
    SCHED -->|chat model, tensor-split| GPU1
    SCHED -->|EMBED| GPU1
    SCHED -->|EXTRACT, CPU-only| CPU
    HA -.memory calls.-> EXTRACT
    EXTRACT --> QDRANT
    EMBED --> QDRANT
```

**Why this shape:** the chat model needs GPU throughput, the embedder is
small enough to live on GPU for free, and the extraction model was
deliberately pinned to CPU/RAM (`num_gpu 0`) so it can never compete with
the chat model for VRAM — see [diagram 05](05-mem0-memory-pipeline.md).

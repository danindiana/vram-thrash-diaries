# 16 — mem0 Architecture: OSS vs. Platform Mode

<img src="graphviz/png/16-mem0-architecture.png" alt="16 — mem0 Architecture: OSS vs. Platform Mode (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/16-mem0-architecture.svg) · [PNG](graphviz/png/16-mem0-architecture.png) · [DOT source](graphviz/16-mem0-architecture.dot)*

[Diagram 05](../05-mem0-memory-pipeline.md) shows the OSS-mode pipeline this
session actually built and fixed. This diagram zooms out one level, to the
choice mem0 itself offers between running entirely local vs. using its
hosted platform — and why OSS mode was the right call here.

```mermaid
flowchart TB
    subgraph oss["OSS mode (used this session)"]
        O1["Self-hosted LLM role\n(mem0-extractor-cpu)"]
        O2["Self-hosted embedder\n(nomic-embed-text)"]
        O3["Local vector store\n(Qdrant, embedded, on-disk)"]
        O4["No external API calls,\nno account required"]
        O1 --> O3
        O2 --> O3
        O3 --> O4
    end

    subgraph platform["Platform mode (not used here)"]
        P1["mem0 hosted API\n(api.mem0.ai)"]
        P2["Managed vector store\n+ managed extraction pipeline"]
        P3["Requires API key,\ndata leaves the box"]
        P1 --> P2 --> P3
    end

    CH(["hermes.config: memory.provider = mem0"])
    CH --> O1
    CH -.not chosen.-> P1
```

**Why OSS mode fit here:** the whole point of this stack is local-first
inference — routing memory extraction through a cloud API would undercut
that for a component that's arguably more sensitive than chat traffic
(memory persists). The tradeoff, documented in the main writeup, is that
you then own the resource-placement problem yourself (where does the
extraction model run, on what device) — which is exactly what the
`mem0-extractor-cpu` fix in [diagram 05](../05-mem0-memory-pipeline.md)
had to solve.

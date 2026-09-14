# 28 — Model Storage & Blob Deduplication

<img src="graphviz/png/28-model-storage-layout.png" alt="28 — Model Storage & Blob Deduplication (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/28-model-storage-layout.svg) · [PNG](graphviz/png/28-model-storage-layout.png) · [DOT source](graphviz/28-model-storage-layout.dot)*

Two derived tags got created this session (`dler-r1-7b-64k`,
`mem0-extractor-cpu`). Neither one duplicated multi-gigabyte weight files
on disk — this is why.

```mermaid
flowchart TD
    TAG1["dler-r1-7b:latest\n(manifest)"] --> BLOB1["blob sha256-1fb8a1f7...\n(the actual weights, 8.1GB)"]
    TAG2["dler-r1-7b-64k:latest\n(manifest, derived)"] -->|"SAME blob,\nreused via content hash"| BLOB1
    TAG2 --> BLOB2["blob sha256-...\n(the num_ctx=65536 layer,\nsmall — just config)"]

    BLOB1 -.-> NOTE["This is why creating dler-r1-7b-64k\nand mem0-extractor-cpu this session\ncost ~0 extra disk — only new config\nlayers were written, weights were\nshared by hash"]

    subgraph maint["Maintenance"]
        M1["ollama list — see all tags\n+ sizes (this box: 40+ models)"]
        M2["unused blobs pruned on\nollama restart UNLESS\nOLLAMA_NOPRUNE is set"]
    end
```

**Why this matters for a box with 40+ pulled tags:** Ollama's manifest/blob
split (content-addressed by sha256, same as Docker/OCI images) means
"derive a variant" is nearly free as long as you're only changing
Modelfile *parameters* — the moment you change the base weights themselves,
you're pulling a genuinely new multi-gigabyte blob, which is the real
disk-cost line to watch when experimenting with model tags.

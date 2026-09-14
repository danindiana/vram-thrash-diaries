# 24 — GGUF File Format Anatomy

<img src="graphviz/png/24-gguf-file-anatomy.png" alt="24 — GGUF File Format Anatomy (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/24-gguf-file-anatomy.svg) · [PNG](graphviz/png/24-gguf-file-anatomy.png) · [DOT source](graphviz/24-gguf-file-anatomy.dot)*

`ollama show <model>` output quoted throughout this repo (`qwen3.context_length`,
`general.license`, etc.) is reading straight out of a GGUF file's key-value
metadata block. This is what's actually inside that file, one layer below
the Modelfile in [diagram 23](23-modelfile-anatomy.md).

```mermaid
flowchart TD
    MAGIC["Header\nmagic bytes 'GGUF', version,\ntensor count, kv count"] --> KV["Key-Value metadata\narchitecture, context_length,\nembedding_length, block_count,\nlicense, tokenizer, chat_template ..."]
    KV --> TINFO["Tensor info table\nname, shape, dtype,\noffset per tensor"]
    TINFO --> TDATA["Tensor data\n(the actual weights,\nquantized per-block)"]

    subgraph seen["Fields seen this session (via 'ollama show')"]
        S1["qwen3.context_length = 262144\nqwen3.block_count = 36"]
        S2["general.license = apache-2.0"]
        S3["tokenizer.chat_template\n(drives TEMPLATE in the Modelfile)"]
    end

    KV -.-> S1
    KV -.-> S2
    KV -.-> S3
```

**Why GGUF specifically, and not raw PyTorch/safetensors:** the KV-metadata
block is self-describing — a runtime like `llama.cpp`/Ollama can read
`context_length`, `block_count`, and the chat template straight from the
file with no external config needed. That's exactly the mechanism behind
`ollama show` printing accurate specs for models this session never wrote
any config for.

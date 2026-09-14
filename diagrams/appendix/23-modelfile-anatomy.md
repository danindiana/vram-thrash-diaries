# 23 — Ollama Modelfile Anatomy

<img src="graphviz/png/23-modelfile-anatomy.png" alt="23 — Ollama Modelfile Anatomy (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/23-modelfile-anatomy.svg) · [PNG](graphviz/png/23-modelfile-anatomy.png) · [DOT source](graphviz/23-modelfile-anatomy.dot)*

Every derived tag this session created (`dler-r1-7b-64k`,
`mem0-extractor-cpu`) came from the same five-directive shape. This is that
shape, generalized, plus exactly what each derived tag used it for.

```mermaid
flowchart TD
    FROM["FROM &lt;blob or tag&gt;\nbase weights (required)"] --> TEMPLATE["TEMPLATE \"...\"\nchat-format string\n(im_start/im_end, roles)"]
    TEMPLATE --> SYSTEM["SYSTEM \"...\"\ndefault system prompt"]
    SYSTEM --> PARAMETER["PARAMETER num_ctx / num_gpu /\ntemperature / stop / ...\nruntime knobs baked into the tag"]
    PARAMETER --> LICENSE["LICENSE \"...\"\nembedded license text"]

    subgraph used["Used this session"]
        U1["dler-r1-7b-64k:\nPARAMETER num_ctx 65536\n(fix a context-window bug)"]
        U2["mem0-extractor-cpu:\nPARAMETER num_gpu 0\n(force CPU-only placement)"]
    end

    PARAMETER -.-> U1
    PARAMETER -.-> U2
```

**The pattern worth remembering:** `ollama show <tag> --modelfile` dumps an
existing tag's full Modelfile, ready to pipe into a new one with one
directive changed. Both derived tags this session used exactly that
workflow — export, edit one `PARAMETER` line, `ollama create` under a new
name. No need to hand-write a Modelfile from scratch for a small tweak.

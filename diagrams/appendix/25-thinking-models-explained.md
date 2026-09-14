# 25 — Reasoning ("Thinking") Models Explained

<img src="graphviz/png/25-thinking-models-explained.png" alt="25 — Reasoning (Thinking) Models Explained (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/25-thinking-models-explained.svg) · [PNG](graphviz/png/25-thinking-models-explained.png) · [DOT source](graphviz/25-thinking-models-explained.dot)*

`qwen3:4b-thinking-2507-q8_0` was this session's daily driver for a long
stretch. This is what the "-thinking" actually means, and the exact hard
error a *different* model in the rotation hit for not supporting it.

```mermaid
flowchart TD
    A["Prompt sent to a\n'-thinking' model variant"] --> B["Model emits a reasoning trace\nfirst (often in &lt;think&gt;...&lt;/think&gt;\nor a separate reasoning_content field)"]
    B --> C["Trace consumes real tokens\n+ real generation time\nbefore the 'real' answer starts"]
    C --> D["Final answer emitted\nafter the trace"]

    subgraph gotcha["Gotcha hit this session"]
        G1["hermes3:8b-llama3.1-q5_K_M\ndoes NOT support the\n'thinking' request parameter"]
        G2["Non-retryable 400 error:\n'does not support thinking'"]
        G1 --> G2
    end

    D -.->|"not every model in\na family supports it"| G1
```

**The practical takeaway:** "thinking" isn't a universal on/off toggle a
harness can safely request from any model — it's a capability specific to
how a given checkpoint was trained and how its chat template is written.
Requesting it from a model that doesn't support it isn't a graceful
degradation, it's a hard 400 error, which is exactly what forced this
session off `hermes3:8b-q5_K_M` mid-stream.

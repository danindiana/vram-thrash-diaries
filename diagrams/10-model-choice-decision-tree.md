# 10 — Model-Choice Decision Tree

<img src="graphviz/png/10-model-choice-decision-tree.png" alt="10 — Model-Choice Decision Tree (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/10-model-choice-decision-tree.svg) · [PNG](graphviz/png/10-model-choice-decision-tree.png) · [DOT source](graphviz/10-model-choice-decision-tree.dot)*

The chat model changed several times this session, each for a concrete,
diagnosable reason — not just preference. This is that chain, and where it
currently dead-ends (deliberately unresolved).

```mermaid
flowchart TD
    Start["dler-r1-7b:latest"] -->|"32K context < 64K minimum"| Fixed["dler-r1-7b-64k\n(derived tag, num_ctx=65536)"]
    Fixed -->|"session-only switch, not persisted"| Q4["qwen3:4b-thinking-2507-q8_0"]
    Q4 -->|"malformed tool_call bridge repeatedly"| BridgeOff["tools.tool_search: off\n(bridge removed — see diagram 08)"]

    Prior["hermes3:8b-llama3.1-q5_K_M\n(tried earlier same session)"] -->|"'does not support thinking'\nnon-retryable error"| Q4

    BridgeOff --> Open{"Open question:\nwhat should the REAL\ndaily-driver default be?"}
    Open -->|"tool-format-matched,\nsame lineage as Hermes itself,\nbut aging"| H8["hermes3:8b-llama3.1-q8_0"]
    Open -->|"strong benchmarks,\nbut documented real-world\nagentic reliability gap"| Ornith["ornith-1.5:9b\n(not pulled/trialed yet)"]

    style Open fill:#fff3e0,stroke:#e65100
    style H8 fill:#e3f2fd,stroke:#1565c0
    style Ornith fill:#e3f2fd,stroke:#1565c0
```

**Recommendation on record:** don't swap to Ornith on benchmark numbers
alone — trial it in a sandboxed profile against real logged tasks first,
since the one hands-on report available shows it failing on exactly the
open-ended, multi-step tool-calling shape this workload is made of.

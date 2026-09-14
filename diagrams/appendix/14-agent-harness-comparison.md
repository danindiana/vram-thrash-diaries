# 14 — Local Agent Harness Comparison

<img src="graphviz/png/14-agent-harness-comparison.png" alt="14 — Local Agent Harness Comparison (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/14-agent-harness-comparison.svg) · [PNG](graphviz/png/14-agent-harness-comparison.png) · [DOT source](graphviz/14-agent-harness-comparison.dot)*

Both harnesses used this session — Hermes Agent (the primary driver
throughout) and Qwen Code (installed near the end) — sit on top of the same
Ollama OpenAI-compatible endpoint but structure their tool-calling
differently underneath.

```mermaid
flowchart TB
    API(["OpenAI-compatible /v1\n(Ollama exposes this)"])

    subgraph hermes["Hermes Agent"]
        H1["General-purpose:\nchat, memory (mem0), skills,\nkanban, plugins"]
        H2["tool_call batching bridge\n(deferred/MCP tools)"]
        H3["Persistent memory\nbuilt in (mem0/holographic)"]
        H1 --> H2 --> H3
    end

    subgraph qwen["Qwen Code"]
        Q1["Coding-focused:\nsubagents, MCP, auto-memory"]
        Q2["Direct tool calls,\nno batching bridge observed"]
        Q3["Config via ~/.qwen/.env\nor settings.json"]
        Q1 --> Q2 --> Q3
    end

    subgraph generic["The general shape (any harness)"]
        G1["Tool loop: model emits\nstructured call → harness\nexecutes → result fed back"]
    end

    API --> H1
    API --> Q1
    H3 -.-> G1
    Q3 -.-> G1
```

**The concrete difference that bit this session:** Hermes's deferred-tool
bridge adds an indirection layer (`tool_call{calls:[...]}`) that small
models kept malforming — see [diagram 08](../08-tool-calling-bridge-fix.md).
Qwen Code's more direct tool-call path never hit that failure mode in the
brief comparison run here, though it hasn't been stress-tested the way
Hermes has across this whole session.

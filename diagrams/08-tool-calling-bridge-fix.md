# 08 — Tool-Calling Bridge Fix

<img src="graphviz/png/08-tool-calling-bridge-fix.png" alt="08 — Tool-Calling Bridge Fix (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/08-tool-calling-bridge-fix.svg) · [PNG](graphviz/png/08-tool-calling-bridge-fix.png) · [DOT source](graphviz/08-tool-calling-bridge-fix.dot)*

Hermes Agent defers non-core tools behind a batching "bridge" tool by
default. Small local models kept malforming it — here's the shape of the
problem and the fix.

```mermaid
flowchart TB
    subgraph Before["BEFORE — bridge enabled (tools.tool_search: auto)"]
        direction TB
        M1["Model wants to call mem0_search\n(a CORE tool, never deferred)"]
        M1 --> W1["Model wraps it in tool_call anyway\n{name: mem0_search, ...}"]
        W1 --> ERR1["Error: 'mem0_search' is not\na deferrable tool"]

        M2["Model wants to batch-call\na deferred tool"]
        M2 --> W2["Sends malformed shape\n(missing 'calls' array)"]
        W2 --> ERR2["Error: tool_call requires\n'calls' (array of {name, arguments})"]
        ERR2 -.repeats identically.-> W2
    end

    subgraph After["AFTER — tools.tool_search: off"]
        direction TB
        M3["Model wants to call\nany tool, core or deferred"] --> D3["Tool appears directly\nin the model-facing list"]
        D3 --> OK["Called directly — no bridge,\nno batching shape to get wrong"]
    end

    Before -->|"config.yaml:\ntools.tool_search.enabled: off"| After
```

**Tradeoff accepted:** deferred/MCP tools now all appear eagerly in the
tool schema instead of being paged in on demand — a larger prompt footprint
if a big MCP toolset is ever added, acceptable at this context size.

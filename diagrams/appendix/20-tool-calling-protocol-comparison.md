# 20 — Tool-Calling Protocol Comparison

<img src="graphviz/png/20-tool-calling-protocol-comparison.png" alt="20 — Tool-Calling Protocol Comparison (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/20-tool-calling-protocol-comparison.svg) · [PNG](graphviz/png/20-tool-calling-protocol-comparison.png) · [DOT source](graphviz/20-tool-calling-protocol-comparison.dot)*

Three real tool-calling wire formats, side by side — not an opinion piece,
just the shapes. [Diagram 08](../08-tool-calling-bridge-fix.md) documents
exactly where the Hermes variant's extra batching layer broke down with
small local models this session.

```mermaid
flowchart TB
    subgraph openai["OpenAI function-calling"]
        O1["tools: [{type:function,\nfunction:{name,parameters}}]"]
        O2["response.tool_calls[]\n{id, function:{name,arguments}}"]
        O1 --> O2
    end

    subgraph hermes["Hermes ChatML tool_call"]
        H1["&lt;tools&gt; JSON schema list\nembedded in system prompt"]
        H2["&lt;tool_call&gt;{name, arguments}\n&lt;/tool_call&gt; inline in text"]
        H3["Deferred tools: batched via\ntool_call{calls:[{name,args}]}\n— the exact shape small models\nmalformed this session"]
        H1 --> H2 --> H3
    end

    subgraph anthropic["Anthropic tool-use"]
        A1["tools: [{name, input_schema}]"]
        A2["content block type=tool_use\n{id, name, input}"]
        A3["Result sent back as\ntype=tool_result content block"]
        A1 --> A2 --> A3
    end

    O2 -.-> NOTE
    H3 -.-> NOTE
    A3 -.-> NOTE
    NOTE["Same underlying idea everywhere: structured\ncall out, structured result back in. Differences\nare syntax + how batching/deferral works — and\nthat's exactly where small models break first."]
```

**Why this matters beyond trivia:** every one of these protocols asks the
model to emit *exact* structured syntax with zero tolerance for drift. A
model that's otherwise perfectly capable at reasoning can still fail an
agentic task purely on protocol fidelity — which is precisely what
happened this session, twice, with two different small models on two
different failure shapes (a missing-thinking-support hard error, and a
malformed batch array).

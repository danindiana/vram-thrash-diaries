# 27 — Local LLM Observability Practices

<img src="graphviz/png/27-observability-practices.png" alt="27 — Local LLM Observability Practices (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/27-observability-practices.svg) · [PNG](graphviz/png/27-observability-practices.png) · [DOT source](graphviz/27-observability-practices.dot)*

Every diagnosis in the main writeup came from one of these three layers.
This is the general map — worth keeping as a checklist independent of the
specific bugs this session hit.

```mermaid
flowchart LR
    subgraph daemon["Daemon layer"]
        D1["journalctl -u ollama\nload/evict/fit events"]
        D2["ollama ps\nwhat's resident RIGHT NOW"]
        D3["nvidia-smi\nactual VRAM + utilization"]
    end

    subgraph agent["Agent layer"]
        A1["~/.hermes/logs/agent.log\nINFO+, tool calls, turns"]
        A2["~/.hermes/logs/errors.log\nWARNING+, the ones that matter"]
        A3["hermes insights\naggregate usage stats"]
    end

    subgraph system["System layer"]
        S1["ss -tlnp\nwhat's actually listening"]
        S2["systemctl list-units\nwhat's actually running"]
    end

    D1 --> OUT["The finding that mattered\nmost this session was\ncross-referencing D1 timestamps\nagainst A1"]
    A1 --> OUT
    S1 -.->|"found the stray\nsecond daemon"| OUT
```

**The habit worth adopting:** none of these three layers alone would have
found the eviction-thrashing root cause. It took the daemon layer's
eviction timestamps lined up against the agent layer's memory-shutdown
log lines to actually prove causation instead of correlation — and it took
the system layer, almost as an afterthought, to find a completely
unrelated stray daemon nobody was looking for.

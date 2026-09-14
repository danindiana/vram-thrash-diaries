# 30 — Prompt / Context Caching Mechanics

<img src="graphviz/png/30-prompt-context-caching.png" alt="30 — Prompt / Context Caching Mechanics (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/30-prompt-context-caching.svg) · [PNG](graphviz/png/30-prompt-context-caching.png) · [DOT source](graphviz/30-prompt-context-caching.dot)*

The very first `journalctl` snippet in [diagram 13](13-ollama-scheduler-internals.md)'s
source material contains a log line about "looking for better prompt" that
this diagram finally explains — and it ties directly back into why the
eviction thrashing documented in the main writeup was worse than it looked.

```mermaid
flowchart TD
    A["Turn 1: full prompt processed,\nKV-cache built token-by-token\n(the slow 'prefill' pass)"] --> B["KV-cache for the SHARED prefix\nkept in memory (system prompt,\nprior turns)"]
    B --> C["Turn 2: only the NEW tokens\nneed prefill — prefix reused"]
    C --> D["Big latency win on long,\nmulti-turn conversations"]

    subgraph seen["Seen this session"]
        S1["Ollama log line: 'looking for\nbetter prompt, base f_keep, f_sim'\n— this IS the prefix-cache lookup"]
        S2["prompt cache is enabled,\nsize limit: 8192 MiB\n(seen in load_model logs)"]
    end

    B -.-> S1

    D -.-> NOTE["Cache is PER LOADED MODEL — another\nreason eviction thrashing was so\ncostly: every reload throws the\nprefix cache away too"]
```

**The compounding cost this explains:** the 313 nemotron evictions in 3
hours weren't just paying the multi-gigabyte reload cost documented in the
main writeup — every one of those reloads also threw away whatever prefix
cache had built up, meaning the *next* request after a reload paid full
prefill cost too. The eviction number alone undersold how bad the
thrashing actually was.

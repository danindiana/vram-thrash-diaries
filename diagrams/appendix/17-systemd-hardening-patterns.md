# 17 — systemd Drop-in Hardening Patterns

<img src="graphviz/png/17-systemd-hardening-patterns.png" alt="17 — systemd Drop-in Hardening Patterns (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/17-systemd-hardening-patterns.svg) · [PNG](graphviz/png/17-systemd-hardening-patterns.png) · [DOT source](graphviz/17-systemd-hardening-patterns.dot)*

Generalized from the exact mess this session found in
`/etc/systemd/system/ollama.service.d/` and fixed — useful for any
systemd service with more than one drop-in file, not just Ollama.

```mermaid
flowchart TD
    subgraph bad["Anti-pattern (found this session)"]
        B1["local-only.conf\nsets MAX_LOADED_MODELS=2"]
        B2["override.conf\nALSO sets MAX_LOADED_MODELS=2\n(+ different NUM_PARALLEL)"]
        B3["Which wins? Resolved silently\nby alphabetical file order —\nnot documented, easy to forget"]
        B1 --> B3
        B2 --> B3
    end

    subgraph good["Pattern applied"]
        G1["One override.conf,\nall variables in one place"]
        G2["Old files renamed\n.disabled-&lt;timestamp&gt;, not deleted\n(reversible)"]
        G3["daemon-reload + restart,\nthen verify effective env via\n'systemctl show -p Environment'"]
        G1 --> G2 --> G3
    end

    B3 -.consolidate.-> G1
```

**The general lesson:** systemd resolves multiple drop-ins for the same
key by file sort order, silently — no warning, no error, just "whichever
file sorts last for that key wins." That's fine as a mechanism but
terrible as an ambient assumption. If you have more than one `.conf` in a
`*.service.d/` directory, audit for overlapping keys before debugging
anything else.

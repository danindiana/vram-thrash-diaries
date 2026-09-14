# 29 — Local vs. Cloud Inference: Decision Framework

<img src="graphviz/png/29-local-vs-cloud-decision.png" alt="29 — Local vs. Cloud Inference: Decision Framework (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/29-local-vs-cloud-decision.svg) · [PNG](graphviz/png/29-local-vs-cloud-decision.png) · [DOT source](graphviz/29-local-vs-cloud-decision.dot)*

This whole repo is about making local inference work *well*. It's still
worth having an honest framework for when local is the wrong call —
this session's own usage history includes real escalations to cloud.

```mermaid
flowchart TD
    Start["New task arrives"] --> Q1{"Does data need to\nstay on-box?"}
    Q1 -->|"yes, must stay local"| Q2{"Is task within local\nmodels' real capability?\n(see appendix 15, 25)"}
    Q1 -->|"no constraint"| CLOUD["Escalate to cloud\n(this session: claude-sonnet-4-6\nused twice per prior insights)"]
    Q2 -->|yes| Q3{"Is latency/throughput\nacceptable at current\nCPU/GPU split?"}
    Q2 -->|"no — needs\nmore capability"| CLOUD
    Q3 -->|yes| LOCAL["Run local\n(this session's whole stack)"]
    Q3 -->|"too slow — fix infra\nfirst, or escalate once"| CLOUD

    LOCAL -.-> NOTE["Sunk-cost trap to avoid: 'we already\nfixed the GPU thrashing, so it must be\nworth running locally' isn't a reason\non its own"]
```

**Why this framework, not a blanket rule:** this repo just spent 20+
diagrams making the case for careful local infra work, which makes it easy
to slide into treating "run it locally" as the goal rather than a means.
The actual goal is getting the task done well — data locality and genuine
capability fit are real reasons to stay local; having already sunk effort
into the GPU scheduler is not.

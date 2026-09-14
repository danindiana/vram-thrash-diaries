# 21 — Embeddings & Vector Search Fundamentals

<img src="graphviz/png/21-embeddings-vector-search.png" alt="21 — Embeddings & Vector Search Fundamentals (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/21-embeddings-vector-search.svg) · [PNG](graphviz/png/21-embeddings-vector-search.png) · [DOT source](graphviz/21-embeddings-vector-search.dot)*

`nomic-embed-text` shows up throughout this repo as "the small GPU model"
without much explanation of what it's actually doing. This is that
explanation, generalized beyond this specific stack.

```mermaid
flowchart TD
    A["Text chunk\n(a memory, a fact, a doc)"] --> B["Embedder model\nnomic-embed-text (this stack)"]
    B --> C["Fixed-length vector\n768 dimensions here"]
    C --> D["Stored in vector index\n(Qdrant, this stack)"]

    E["Query text → same embedder\n→ query vector"] --> F["Approximate Nearest Neighbor\nsearch (cosine similarity)"]
    D --> F --> G["Top-K closest vectors\nreturned as 'relevant memories'"]

    G -.-> NOTE["Rule: query and stored vectors MUST come\nfrom the SAME embedder — swapping embedding\nmodels invalidates the whole existing index"]
```

**Why this rule matters in practice:** if you ever change `mem0.json`'s
embedder (say, to a different-dimension model), every memory written under
the old embedder becomes unsearchable — not corrupted, just geometrically
incomparable to new query vectors. Re-embedding the whole store is the only
fix, which is worth knowing *before* you casually swap an embedder model.

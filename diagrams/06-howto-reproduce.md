# 06 — How To: Diagnose and Fix This on Your Own Box

<img src="graphviz/png/06-howto-reproduce.png" alt="06 — How To: Diagnose and Fix This on Your Own Box (dark/neon)" width="100%">

*Dark/neon render ([Graphviz](https://graphviz.org)): [SVG](graphviz/svg/06-howto-reproduce.svg) · [PNG](graphviz/png/06-howto-reproduce.png) · [DOT source](graphviz/06-howto-reproduce.dot)*

A generic checklist, not tied to any specific hostname or model — follow
this if `ollama ps` looks wrong or a model seems to load-then-vanish.

```mermaid
flowchart TD
    S["Symptom: eviction thrashing,\nor 'GPU idle but model not on it'"] --> A["Check ollama ps\nrepeatedly over ~30s"]
    A --> B{"Same model\ncoming and going?"}
    B -->|yes| C["journalctl -u ollama --since '-1h'\ngrep 'loaded runners\\|make room'"]
    B -->|no, nothing ever loads| D["Check for a SECOND ollama daemon:\nss -tlnp | grep ollama\nsystemctl list-units | grep -i ollama"]

    C --> E["Count distinct concurrent roles needed:\nchat model + memory/extraction LLM + embedder + ..."]
    E --> F{"count > OLLAMA_MAX_LOADED_MODELS?"}
    F -->|yes| G["Either raise the cap,\nor move a role off-GPU (num_gpu=0 Modelfile param)"]
    F -->|no| H["Check OLLAMA_MAX_LOADED_MODELS semantics\nin YOUR ollama version's 'serve --help' —\ndon't trust old comments/docs"]

    D --> I{"Found a second daemon\non a different port?"}
    I -->|yes| J["Check its CUDA_VISIBLE_DEVICES\nagainst nvidia-smi -L —\nis it double-booking a real GPU?"]
    I -->|no| K["Check systemd drop-ins for\nconflicting/overlapping .conf files"]

    G --> V["Verify: reload runners together,\nconfirm 0 eviction over a few minutes"]
    J --> V
    K --> V
    H --> V
```

**Concrete commands used throughout this session:**
```bash
ollama ps
journalctl -u ollama --since "-3 hours" --no-pager | grep -E 'loaded runners|make room'
journalctl -u ollama --since "-3 hours" --no-pager | grep -oE "runner.name=registry.ollama.ai/library/[^ ]+" | sort | uniq -c | sort -rn
ss -tlnp | grep -i ollama
systemctl list-units --type=service | grep -i ollama
nvidia-smi -L
cat /etc/systemd/system/ollama.service.d/*.conf
```

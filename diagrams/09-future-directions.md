# 09 — Future Directions

What's deliberately left open at the end of this session.

```mermaid
mindmap
  root((Open items))
    Chat model choice
      hermes3:8b-llama3.1-q8_0 — tool-format-matched, but aging
      Ornith-1.5-9B — strong benchmarks, documented real-world tool-use gap
      Not yet decided — current default is a bugfix-of-convenience tag
    Qwen Code
      Installed and wired to local Ollama, confirmed working
      Not yet compared head-to-head against Hermes Agent on real tasks
      qwen3-coder:30b-a3b — 46GB, partial CPU offload — lighter tag not pulled
    Billing
      OpenRouter and Nous auxiliary providers marked unhealthy
      "payment/credit error" — account-level, out of scope for local fixes
    Session restart
      Config changes this session (tool_search off, mem0-extractor-cpu,
      MAX_LOADED_MODELS=3) not yet live on any already-running Hermes session
      Needs a manual relaunch to pick up
    Cleanup
      Stray ollama-gpu1.service disabled, unit file kept (not deleted)
      Old systemd drop-ins kept as .disabled/.superseded backups, not removed
```

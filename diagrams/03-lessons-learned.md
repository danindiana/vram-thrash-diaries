# 03 — Lessons Learned

The generalizable takeaways — useful even if you never touch Ollama or
Hermes Agent specifically.

```mermaid
mindmap
  root((Lessons learned))
    Observability
      "ollama ps" only shows the daemon it's talking to
      A second daemon on a different port is invisible until you go looking
      journalctl eviction counts are the real signal, not vibes
      "GPU idle" and "nothing loaded" can both be true and still be wrong
    Config hygiene
      Two overlapping systemd drop-ins silently resolved by file sort order
      A documented env var isn't necessarily a live one in your version
      "serve --help" is the source of truth, not old comments in a .conf file
    Resource caps are blunt instruments
      A global "max loaded models" cap doesn't know CPU-only doesn't need a GPU slot
      Fixing one contention (desktop vs GPU) can starve something else (memory extraction)
      Small, cheap, correctly-placed models beat "just cap harder"
    Small models and structured protocols
      Benchmarks that look great can still fail on YOUR exact workload shape
      Tool-call batching layers are exactly where small models break first
      Sometimes the fix is removing the indirection, not upgrading the model
    Process hygiene
      A daemon can run unnoticed for hours if nothing ever queries its port
      Deleting a file a live process holds open is safe on Linux — but isn't a fix
      Verify a hypothesis (fuser on the file, not the directory) before acting on it
```

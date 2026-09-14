# Session 1789426452 — Hermes `/goal` Judge Outage + Nemotron Context Regression: Diagnosis & Fix

**Follow-on to:** [[hermes-agent-ollama]], `session_1789414749` (why `/goal` uses a judge model)
**Machine:** the-box
**Outcome:** Both issues fixed and verified live.

## 1. Trigger

The user's `/goal` loop paused with:
```
⏸ Goal paused — judge API returned errors (5 turns). Check the goal_judge provider/key in
~/.hermes/config.yaml:
  auxiliary:
    goal_judge:
      provider: deepseek
      model: deepseek-flash
Then /goal resume to continue.
```

**First correction made during investigation:** the `deepseek`/`deepseek-flash` text in that
message is **not** the actual broken config — it's a hardcoded example baked into Hermes's own
remediation-hint string (`hermes_cli/goals.py:1059`,
`_JUDGE_CONFIG_HINT.format(provider="deepseek", model="deepseek-flash")`). It always shows that
literal example regardless of what's actually configured. This cost real debugging time until the
actual `~/.hermes/config.yaml` and `~/.hermes/logs/agent.log` were read directly.

## 2. Root Cause: A Two-Layer Config Rot

Direct log evidence (`~/.hermes/logs/agent.log`):
```
14:10  Auxiliary goal_judge: using custom (ornith-1.5:9b) at http://localhost:11434/v1/
14:10  goal judge: API call failed (404 — "model 'ornith-1.5:9b' not found") — falling through to continue
...
14:43  Auxiliary goal_judge: using custom (anthropic/claude-opus-4.6) at http://localhost:11434/v1/
14:43  goal judge: API call failed (404 — "model 'anthropic/claude-opus-4.6' not found") — falling through to continue
...repeats every turn through 16:45...
```

Two stacked problems:
1. `auxiliary.goal_judge.model` was `ornith-1.5:9b`, but that model had since been **removed from
   the local Ollama store** (`ollama list` confirmed no `ornith-1.5` present).
2. Sometime after 14:33, `goal_judge.model` was changed to `anthropic/claude-opus-4.6` — a real
   Anthropic cloud model name — while `provider` stayed `custom`, which routes to the **local
   Ollama endpoint** (`http://localhost:11434/v1`) with no override. Ollama obviously has no such
   model, so every call 404'd identically. Logs show the running agent itself web-searching
   `"hermes agent goal_judge provider deepseek-flash config.yaml error"` around 14:13 — plausibly
   a self-repair attempt that produced this bad value, though this was not conclusively proven.

Each 404 falls through to `verdict="continue"` (fail-open by design — see `session_1789414749`),
but `consecutive_transport_failures` still increments per call. At **5 consecutive transport
failures** (`DEFAULT_MAX_CONSECUTIVE_TRANSPORT_FAILURES`), the loop auto-paused, printing the
generic hint text quoted above.

## 3. Fix Path Chosen: Anthropic Provider via `claude setup-token`

Rejected repairing `goal_judge` back onto local Ollama, since:
- The originally-pinned local model (`ornith-1.5:9b`) no longer exists.
- The user specifically wanted an **Anthropic API-accessible model** for judging — independent of
  whatever local model churn happens to the main chat model.

Deliberately **did not** touch `~/.claude/.credentials.json` (Claude Code's own session
credential store) — that's not a portable API key meant for third-party apps. Instead:

1. User ran `claude setup-token` (a sanctioned Claude Code command — *"Set up a long-lived
   authentication token, requires Claude subscription"*) and provided the resulting token.
2. Token added to `~/.hermes/.env` as `CLAUDE_CODE_OAUTH_TOKEN=<token>` (`chmod 600`, value never
   echoed to any log/terminal output during setup).
3. `~/.hermes/config.yaml` `auxiliary.goal_judge` block changed:
   ```yaml
   # before (broken)
   goal_judge:
     provider: custom
     model: anthropic/claude-opus-4.6

   # after (fixed)
   goal_judge:
     provider: anthropic
     model: claude-haiku-4-5-20251001
   ```
   Hermes's native `anthropic` provider plugin
   (`hermes-agent/plugins/model-providers/anthropic/__init__.py`) accepts
   `ANTHROPIC_API_KEY`, `ANTHROPIC_TOKEN`, or `CLAUDE_CODE_OAUTH_TOKEN` and defaults its aux model
   to `claude-haiku-4-5-20251001` — fast, cheap, reliable at strict JSON output, a good fit for a
   per-turn judge call.

### Verification gotcha: wrong Python interpreter

First verification attempt used system `python3` (3.10.12) and failed with
`ModuleNotFoundError: No module named 'tomllib'` deep in an unrelated exception-handling path
(`agent/relay_llm.py` → `agent/relay_runtime.py`) — `tomllib` is stdlib-only from Python 3.11+.
Hermes ships its **own bundled venv** at `~/.hermes/hermes-agent/venv/bin/python` (3.11.16,
confirmed via the `hermes` launcher script at `~/.local/bin/hermes`, which explicitly unsets
`PYTHONPATH`/`PYTHONHOME` and execs that exact interpreter). Re-running the verification through
the correct venv:
```python
from agent.anthropic_credentials import resolve_anthropic_token
# -> resolved token present: True, prefix sk-ant-oat01 (confirms CLAUDE_CODE_OAUTH_TOKEN is an
#    OAuth *access* token, not a console API key — Hermes's resolve_anthropic_token() explicitly
#    supports this format)

from agent.auxiliary_client import call_llm
call_llm(task="goal_judge", messages=[...], temperature=0, max_tokens=16, timeout=20)
# -> "OK"  — confirmed working end-to-end
```

### Live process still needs a restart

`ps aux` showed an interactive `hermes` process (PID 1220688, `pts/0`) running since before any of
these fixes — config/env are read once at process start (documented pattern, see
`session_1789341581`/`session_1789371423`). The user was told to restart that session and then run
`/goal resume`.

## 4. Second Issue Found While Verifying: Nemotron Context-Length Regression Risk

While confirming the fix, the user asked: *"our nemotron lightning model's context window went
from 1 million down to 262K?"* Investigated via `ollama ps` and the config-backup history in
`~/.hermes/backups/config/config.yaml.good.*`:

| Backup timestamp | `model.default` | `model.context_length` |
|---|---|---|
| 13:45:56 | `ornith-1.5:9b` | `262144` |
| 15:31:35 | `ornith-1.5:9b` | `262144` |
| 15:36:37 | `nemotron-3.5-lightning:1m` | `262144` ← **stale, not reset** |
| 17:09:24 (this session's goal_judge edit) | `nemotron-3.5-lightning:1m` | `262144` (untouched by that edit) |

Log confirmation of the switch: at `15:32:58`, a turn-context log line reads *"model was just
switched from ornith-1.5:9b to nemotron-3.5-lightning:1m"* — a deliberate `/model`/`hermes model`
switch, unrelated to any of this session's edits. `context_length` simply never got reset back to
nemotron's native `1048576` (leftover from the earlier `ornith-1.5:9b` context-window fix in
session_1789371423) when the default switched back.

**Why this matters:** per the exact mismatch-detection mechanism documented in
session_1789371423 (`agent_init.py::_scope_context_length_to_default_runtime`), Hermes *ignores*
`model.context_length` only when it **doesn't** match the currently active runtime model — and
*applies* it when it does. With `model.default` now `nemotron-3.5-lightning:1m` and
`context_length` still `262144`, the config was **armed to silently clamp nemotron's context from
1,048,576 down to 262,144** on the next fresh load (restart, or the model's own idle-timeout
eviction/reload).

**Verified state at the time of the report:** `ollama ps` still showed
`CONTEXT 1048576` — the live warm runner had not yet been reloaded under the bad config, so no
actual degradation had occurred yet. This was a live risk, not (yet) an active regression.

**Fix:**
```yaml
# before
model:
  context_length: 262144
# after
model:
  context_length: 1048576
```
Also folded into the same pending `hermes` process restart already needed for the goal_judge fix
— one restart now picks up both corrections.

## 5. Bonus Finding: Agent Self-Patch Guard

Log line at `15:36:32`:
```
Tool patch returned error: {"error": "Refusing to write to Hermes config file: ~/.hermes/config.yaml
Agent cannot modify security-sensitive configuration. Edit ~/.hermes/config.yaml directly or use 'hermes config' ..."}
```
Confirms Hermes has a built-in guard preventing its own agent tools from writing directly to
`config.yaml` — whatever produced the bad `anthropic/claude-opus-4.6` value, it wasn't through that
blocked `patch` tool path. Worth remembering as a security control already in place, not something
to route around.

## 6. Final State

| Setting | Before | After |
|---|---|---|
| `auxiliary.goal_judge.provider` | `custom` (→ local Ollama) | `anthropic` |
| `auxiliary.goal_judge.model` | `anthropic/claude-opus-4.6` (invalid on Ollama) | `claude-haiku-4-5-20251001` |
| `~/.hermes/.env` | no Anthropic credential | `CLAUDE_CODE_OAUTH_TOKEN` set (0600 perms) |
| `model.context_length` | `262144` (stale, armed to clamp nemotron) | `1048576` (matches nemotron's native baked context) |

Both fixes verified: judge call returns a real response through the correct venv; `ollama ps`
confirms nemotron's live context is still `1048576`. User needs to restart the interactive
`hermes` session (`pts/0`) once, then `/goal resume`.

## 7. Diagram Index

`diagrams/`, each as `.svg` + `.png`, dark/neon Graphviz style (consistent with
`session_1789414749/diagrams`):

1. `01_incident_timeline` — full incident timeline, pause → diagnosis → fix → second issue → fix
2. `02_judge_failure_chain` — how `custom` provider + missing/wrong model produced repeated 404s
3. `03_log_evidence_timeline` — the exact log-line sequence from 14:10 to auto-pause
4. `04_anthropic_fix_architecture` — new goal_judge call path via the `anthropic` provider plugin
5. `05_credential_resolution_priority` — `resolve_anthropic_token()`'s lookup order
6. `06_verification_steps` — the venv-mismatch gotcha and the corrected verification sequence
7. `07_python_interpreter_gotcha` — system python3.10 vs Hermes's bundled venv python3.11
8. `08_context_length_regression_mechanism` — how a stale config value + a model switch armed a
   silent context clamp
9. `09_config_backup_timeline` — the `config.yaml.good.*` snapshot history showing exactly when
   each field drifted
10. `10_before_after_state` — side-by-side final state comparison (table 6, visualized)

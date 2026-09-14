# Verbose Report: Why Hermes Agent's `/goal` Uses a Judge Auxiliary Model

**Session:** 1789414749
**Machine:** the-box (this host)
**Source examined:** `~/.hermes/hermes-agent/hermes_cli/goals.py`,
`~/.hermes/hermes-agent/hermes_cli/config_defaults.py`
**Method:** direct source read, no assumptions from memory; corroborated against this
workspace's own prior operational session `session_1789184927/SESSION.md`.

---

## 1. Executive Summary

Hermes Agent's `/goal` command turns a single request into a **persistent, multi-turn
autonomous loop**: the agent keeps taking turns toward a stated objective without the
user re-prompting each time. Something has to decide, after every turn, whether the
goal is actually finished, permanently stuck, waiting on something, or should keep
going. Hermes answers that with a second, independently configured LLM call — the
**"goal judge"** — rather than trusting the same model that just did the work to also
grade its own output.

There is no external "Judge API" product. It's an internal auxiliary-model task
(`auxiliary.goal_judge` in `~/.hermes/config.yaml`), routed through the same generic
`call_llm(task=...)` machinery Hermes uses for other background jobs like
`compression`, `approval`, `mcp`, and `curator`. By default it can point at the same
local Ollama endpoint as the main chat model; it is only a separate "API" in the sense
of being a separate *call* with its own prompt, model pin, and timeout — not a
separate service.

## 2. Background: What `/goal` Actually Is

From the module docstring, `hermes_cli/goals.py:1-7`:

> "Persistent session goals — the Ralph loop for Hermes. A goal is a free-form
> objective that stays active across turns; after each turn an auxiliary-model judge
> decides whether it is satisfied. The continuation prompt is a normal user message
> appended via `run_conversation` (no system-prompt mutation or toolset swap — prompt
> caching stays intact). Judge failures are fail-OPEN (`continue`); the turn budget is
> the backstop."

Key mechanics:
- A goal loop runs for up to `DEFAULT_MAX_TURNS = 20` turns (`config_defaults.py`)
  before auto-pausing regardless of judge verdicts — a hard backstop independent of
  judge behavior.
- Continuation between turns is injected as an ordinary user message
  (`CONTINUATION_PROMPT_TEMPLATE`), not a system-prompt change, so the main
  conversation's prompt cache is never invalidated by the goal machinery.
- The loop can carry a **completion contract** (structured "what does done mean")
  and/or user-added **subgoals** (`/subgoal`) that both the continuation prompt and
  the judge see.

## 3. What "Judge API" Resolves To

`hermes_cli/config_defaults.py:734`:
```python
"goal_judge": _aux(60),          # /goal satisfaction + contract drafting; JSON calls
```

This sits in the same `auxiliary` config block as every other background task Hermes
runs off the main chat model: `vision`, `compression`, `skills_hub`, `approval`,
`review`, `mcp`, `title_generation`, `memory_query_rewrite`, `tts_audio_tags`,
`triage_specifier`, `kanban_decomposer`, `profile_describer`, `curator`, `monitor`,
`background_review`, `moa_reference`, `moa_aggregator`. Each of these is a named "slot"
with its own provider/model/timeout, defaulting to `"auto"` (the main model) unless
explicitly pinned in `config.yaml`. `goal_judge` is not special infrastructure — it's
one more named slot in this same table, given a 60-second default timeout by the `_aux(60)`
helper.

There is no dedicated judge binary, container, or remote endpoint. The user (or the
default config) can point `auxiliary.goal_judge.provider`/`.model` at anything Hermes
already supports — local Ollama, OpenRouter, Nous Portal, etc.

## 4. Full Call Sequence

```
turn ends (agent produces last_response)
        │
        ▼
GoalManager.evaluate_after_turn(last_response, ...)      [goals.py]
        │
        ├─ if state.status != "active" → return "inactive"
        ├─ if is_waiting() → quiesce without burning a turn
        ├─ state.turns_used += 1
        │
        ▼
_check_gates()   ── deterministic shell-command gates ──►  if a gate fails:
        │                                                    skip judge entirely,
        │  (all gates pass, or none defined)                 feed gate output back
        ▼                                                    as the continuation
judge_goal(goal, last_response, subgoals, contract, ...)
        │
        ├─ builds the prompt: contract > subgoals > plain (priority order)
        ▼
_call_goal_judge_llm(call_llm, JUDGE_SYSTEM_PROMPT, prompt, timeout)
        │
        ▼
call_llm(task="goal_judge", messages=[...], temperature=0,
         max_tokens=_goal_judge_max_tokens(), timeout=timeout)
        │      (this is where auxiliary.goal_judge.provider/model/extra_body/
        │       reasoning_effort/retries from config.yaml actually get applied)
        ▼
raw JSON reply  →  _parse_judge_response(raw)
        │
        ▼
(verdict, reason, parse_failed, wait_directive, transport_failed)
        │
        ▼
evaluate_after_turn applies verdict to loop state (see §5)
```

Exact call site (`hermes_cli/goals.py:860-873`):
```python
def _call_goal_judge_llm(call_llm, system_prompt, user_prompt, timeout) -> str:
    """Route through call_llm so auxiliary.goal_judge.* config (provider/model, extra_body,
    reasoning_effort, retries) all apply. Returns the raw reply text."""
    resp = call_llm(
        task="goal_judge",
        messages=[{"role": "system", "content": system_prompt},
                  {"role": "user", "content": user_prompt}],
        temperature=0, max_tokens=_goal_judge_max_tokens(), timeout=timeout,
    )
    try:
        return resp.choices[0].message.content or ""
    except Exception:
        return ""
```

Note `temperature=0` — the judge is deliberately run deterministically, unlike the main
chat turn, because it's producing a classification decision, not creative output.

## 5. The Four Verdicts and Their Effects

From `evaluate_after_turn` (`hermes_cli/goals.py:1443-1520`):

| Verdict | Meaning | Effect on loop |
|---|---|---|
| `done` | Goal fully satisfied, deliverable exists | `state.status = "done"`, loop stops, saved |
| `blocked` | Goal genuinely unachievable as stated, or needs user input | Loop **pauses** (not stopped as done) — surfaces the judge's reason, user can `/goal set` to re-scope or `/goal resume` to override |
| `wait` (+ wait_directive) | Parked on a running background process or PID | Loop parks without burning a turn until the process/deadline resolves |
| `continue` (implicit default) | Not yet done | Loop keeps going, next continuation prompt sent, turn budget decremented |

Design comment directly from source (`goals.py:1485-1489`):
> "BLOCKED is NOT done: pause so the user sees the judge's reason and can re-scope or
> override, instead of burning turns on an unachievable goal or waving it through as
> complete."

This is the crux of *why* a judge exists at all: without it, the loop would have no
principled way to distinguish "the agent believes it's done" from "the agent is stuck
and rationalizing," short of a human watching every turn.

## 6. Six Design Reasons for a Separate Judge Call

1. **Structured verdict, not conversation.** `JUDGE_SYSTEM_PROMPT` (`goals.py:113`)
   demands a strict one-line JSON verdict. Running this as a separate call — rather
   than asking the main model to also emit a parseable verdict inline — keeps the main
   conversation's system prompt and toolset untouched, so prompt caching on the main
   chat is never broken by goal-loop bookkeeping.

2. **Independence from the worker.** The model that just produced `last_response` has
   an incentive (explicit or emergent) to consider its own work finished. A separate
   judge — often configured to a different, stricter, or cheaper model — is less prone
   to this self-assessment bias. `_JUDGE_CONFIG_HINT` (`goals.py:1059`) explicitly
   suggests remediation models like `deepseek-flash` or
   `google/gemini-3-flash-preview` when the current judge proves unreliable, implying
   Hermes expects judge and worker models may differ.

3. **Confirmed as a real quality gate in this workspace.** `session_1789184927/SESSION.md`
   (lines 132-139) records the judge rejecting two shallow completion attempts on a
   real overnight research `/goal` run before accepting a genuinely complete one —
   direct evidence the mechanism does what it's designed for, not just theory.

4. **Fail-open with independent failure thresholds.** Two separate counters:
   - `consecutive_parse_failures` (bad/non-JSON judge output) → auto-pause at
     `DEFAULT_MAX_CONSECUTIVE_PARSE_FAILURES = 3`
   - `consecutive_transport_failures` (401/timeout/DNS) → auto-pause at
     `DEFAULT_MAX_CONSECUTIVE_TRANSPORT_FAILURES = 5`
   Source comment (`goals.py:1477-1479`): "Parse failures reset on any usable reply
   INCLUDING transport errors, so a flaky network doesn't trip the auto-pause meant
   for bad judge models; transport failures are counted separately because persistent
   API errors (401, DNS) mean a broken config." A single unreachable judge cannot
   silently hang the loop — it degrades to `"continue"` per turn and eventually
   auto-pauses with an actionable message pointing at the exact config key to fix.

5. **Reused for completion-contract drafting, not just scoring.** `draft_contract()`
   (`goals.py:1019-1044`) uses the *same* `goal_judge` auxiliary slot to expand a
   plain-language objective into a structured `GoalContract` (what counts as done, how
   to verify it, constraints, scope) before the loop even starts. This means the
   "judge model" is a general-purpose structured-reasoning slot for the goal feature,
   not narrowly a post-hoc grader.

6. **Deterministic gates take priority and can bypass the judge entirely.**
   `_check_gates()` runs first; a failing shell-command gate short-circuits the judge
   completely (`config_defaults.py` comment, ~line 48-50): "A failed gate
   short-circuits the judge — its output IS the continuation prompt, so the agent
   works on concrete evidence instead of a vibe check." The LLM judge is a fallback for
   the (common) case where no hard deterministic check is available, not the primary
   mechanism when one is.

## 7. Judge Output Budget Gotcha (Worth Knowing)

`config_defaults.py:34-37`:
```python
# Judge output budget. Reasoning models burn hidden-reasoning tokens before the visible one-line
# JSON verdict; 200 (the original) reliably truncated it and tripped the auto-pause. 4096 covers
# every model live-tested; override via auxiliary.goal_judge.max_tokens.
DEFAULT_JUDGE_MAX_TOKENS = 4096
```
A reasoning-capable judge model spends hidden tokens "thinking" before emitting its
one-line verdict. An earlier, tighter token budget (200) reliably truncated that
output and falsely tripped the parse-failure auto-pause — fixed by raising the default
to 4096. This is the same category of context/output-budget gotcha already documented
for the main chat model in [[hermes-agent-ollama]] (session_1789371423's three-layer
context-window investigation) — it recurs anywhere Hermes talks to a reasoning model.

## 8. Real-World Evidence Walkthrough

From `session_1789184927/SESSION.md`:
- Triggered by `/goal` — user wanted Hermes to research a topic autonomously
  overnight, later switched to chained `--goal` kanban tasks.
- `frontier_map.md` (145 lines): the goal-loop's judge **rejected the first
  completion attempt as too shallow**, forcing a real second pass.
- `novel_paradigms_proposals.md`: the judge **rejected two shallow attempts** before
  accepting a third.

This is the mechanism from §5-6 operating exactly as designed: the worker declared
(or implied) completion, the separate judge call disagreed, and the loop kept going
until the deliverable actually met the bar — with zero human intervention required to
catch the shallow attempts.

## 9. Key Source Locations

| Location | What's there |
|---|---|
| `hermes_cli/goals.py:1-7` | Module purpose docstring |
| `hermes_cli/goals.py:33-46` | Judge timeout/token/failure-threshold constants + rationale comments |
| `hermes_cli/goals.py:113-` | `JUDGE_SYSTEM_PROMPT` |
| `hermes_cli/goals.py:180-` | `JUDGE_USER_PROMPT_TEMPLATE` / subgoals / contract variants |
| `hermes_cli/goals.py:860-873` | `_call_goal_judge_llm` — the actual `call_llm` invocation |
| `hermes_cli/goals.py:876-933` | `judge_goal` — prompt selection + verdict parsing |
| `hermes_cli/goals.py:1019-1044` | `draft_contract` — judge slot reused for contract drafting |
| `hermes_cli/goals.py:1059-1060` | `_JUDGE_CONFIG_HINT` — remediation text shown to the user |
| `hermes_cli/goals.py:1443-1520` | `evaluate_after_turn` — gates-before-judge, verdict → state transitions, auto-pause logic |
| `hermes_cli/config_defaults.py:33-52` | Judge/gate constants and design-rationale comments |
| `hermes_cli/config_defaults.py:734` | `"goal_judge": _aux(60)` — the config slot itself |

## 10. Diagram Index

All diagrams are in `diagrams/`, each rendered as both `.svg` and `.png`:

1. `01_goal_loop_state_machine` — `/goal` loop states: active → judging → done / blocked / wait / paused
2. `02_judge_call_sequence` — turn boundary through gate check, judge_goal, call_llm, to parsed verdict
3. `03_config_resolution_chain` — `config.yaml auxiliary.goal_judge` → `call_llm` task routing → provider/model
4. `04_gates_vs_judge_decision_flow` — deterministic gate short-circuit vs LLM judge fallback
5. `05_failure_handling_autopause` — independent parse-failure and transport-failure counters and thresholds
6. `06_completion_contract_drafting` — objective → `draft_contract` → `GoalContract` → judge's source of truth
7. `07_judge_verdict_outcomes` — the four verdicts and their resulting loop-state transitions
8. `08_prompt_template_selection` — plain / subgoals / contract prompt priority into the judge prompt
9. `09_model_isolation_rationale` — why the judge call is isolated from the main chat model/context/cache
10. `10_real_world_timeline` — `session_1789184927` walkthrough: two rejections, then acceptance

## 11. Bottom Line

`/goal`'s "Judge API" is not a separate product or external dependency. It is a
second, independently configurable LLM call (`auxiliary.goal_judge`) that Hermes makes
after every autonomous turn — using the same generic auxiliary-task machinery as its
other background jobs — specifically so the loop has a structured, temperature-0,
JSON-only, independently-model-able opinion on whether the goal is done, blocked, or
should continue, instead of trusting the same model that just did the work to also
grade its own homework. Deterministic shell-command gates take priority over it when
available; when the judge itself misbehaves (bad JSON, unreachable API), the loop
fails open and auto-pauses with an actionable config hint rather than hanging
indefinitely. This is not theoretical — this workspace has direct evidence of the
judge rejecting shallow work and forcing real retries on a live overnight research
run.

# Session 1789414749 — Why does Hermes Agent need a "Judge API" for `/goal`?

## Question
User asked why Hermes Agent (Nous Research CLI agent, installed on this machine at
`~/.hermes/hermes-agent`, see [[hermes-agent-ollama]]) needs/requests a "Judge API"
when using the `/goal` functionality.

## Method
Read Hermes Agent's own source directly (not memory/docs) at
`~/.hermes/hermes-agent/hermes_cli/goals.py` and
`~/.hermes/hermes-agent/hermes_cli/config_defaults.py`. Corroborated with real
in-the-wild behavior recorded in this workspace's own prior session,
`session_1789184927/SESSION.md`.

## What "Judge API" actually is
There is **no separate external "Judge API" product**. It's Hermes routing an ordinary
chat-completion call through whatever provider/model is configured under
`auxiliary.goal_judge` in `~/.hermes/config.yaml` — the same auxiliary-model machinery
Hermes uses for other background tasks (`compression`, `approval`, `mcp`, `curator`,
etc., see `config_defaults.py` around line 700-740). On this machine, with no override,
that resolves to the same local Ollama endpoint as the main chat model.

## Why `/goal` needs a judge at all

`hermes_cli/goals.py` module docstring:

> "Persistent session goals — the Ralph loop for Hermes. A goal is a free-form
> objective that stays active across turns; after each turn an auxiliary-model judge
> decides whether it is satisfied."

`/goal` is a **persistent, multi-turn autonomous loop** (up to `DEFAULT_MAX_TURNS = 20`
turns). After every turn, something has to decide: is the goal DONE, BLOCKED
(unachievable), WAIT (parked on a background process), or should the loop CONTINUE?
That decision needs a discrete, structured answer — not more free-form chat — so
Hermes offloads it to a dedicated judge call rather than trying to infer it from the
ongoing conversation.

Six concrete reasons found in the code:

1. **Structured verdict, not conversation.** The judge is given a strict system
   prompt (`JUDGE_SYSTEM_PROMPT`, `goals.py:113`) demanding one of four JSON verdicts
   (`DONE` / `BLOCKED` / `wait` / `continue`) plus a reason. Isolating this in its own
   call keeps it out of the main chat context and prompt cache — the main
   conversation's system-prompt/toolset never mutates for judging
   (`goals.py` docstring: "no system-prompt mutation or toolset swap — prompt caching
   stays intact").

2. **Independence from the worker.** Letting the same model that just produced an
   answer also grade its own completion is a weak self-check — it's prone to
   rubber-stamping shallow work. A separate (often stricter/cheaper) model is
   configured specifically to avoid this; `_JUDGE_CONFIG_HINT` in `goals.py:1059`
   suggests routing to e.g. `deepseek-flash` or
   `google/gemini-3-flash-preview` if the current judge model is unreliable.

3. **Confirmed as a real quality gate, not just theory.** This workspace's own prior
   session (`session_1789184927/SESSION.md`, lines 132-139) recorded the judge
   **rejecting shallow completion attempts twice** on real `/goal`-driven research
   tasks and forcing genuine retries — "the quality gate working exactly as
   designed."

4. **Fail-open design, budget as backstop.** Judge parse failures (bad/non-JSON
   output) auto-pause after `DEFAULT_MAX_CONSECUTIVE_PARSE_FAILURES = 3`; judge
   transport failures (401, timeout, DNS) auto-pause after
   `DEFAULT_MAX_CONSECUTIVE_TRANSPORT_FAILURES = 5` (`config_defaults.py`). If the
   judge API is entirely unreachable, the loop still can't run forever — the turn
   budget is the ultimate backstop, and judge errors fall through to `"continue"`
   rather than silently killing the goal.

5. **Reused for contract drafting, not just post-turn scoring.** The same
   `auxiliary.goal_judge` model is also used to expand a free-form goal into a
   structured "completion contract" up front (`_draft_completion_contract`,
   `goals.py:1021`) — defining what "done" means, how to prove it, and scope
   boundaries — so the judge later has a concrete target to check against instead of
   vague intent.

6. **Deterministic gates run first and can skip the judge entirely.** `/goal gate add
   <cmd>` lets a user attach shell commands that must pass before the goal can be
   declared done. A failing gate short-circuits the judge outright — "its output IS
   the continuation prompt, so the agent works on concrete evidence instead of a vibe
   check" (`config_defaults.py` comment, line ~48-50). The judge (an LLM opinion) is
   only consulted when there's no hard deterministic check available.

## Key source locations
- `hermes_cli/goals.py:1-7` — module purpose docstring.
- `hermes_cli/goals.py:113-` — `JUDGE_SYSTEM_PROMPT`.
- `hermes_cli/goals.py:860-933` — `_call_goal_judge_llm` / `judge_goal` (the actual
  call site, routed through `call_llm(task="goal_judge", ...)`).
- `hermes_cli/goals.py:1021` — `_draft_completion_contract` (judge model reused for
  contract drafting).
- `hermes_cli/goals.py:1059` — `_JUDGE_CONFIG_HINT` (points the user at
  `auxiliary.goal_judge.provider/model` in `config.yaml` when the judge is
  unreliable).
- `hermes_cli/config_defaults.py:33-52` — judge timeout/token/failure-threshold
  constants and their rationale comments.
- `hermes_cli/config_defaults.py:734` — `"goal_judge": _aux(60)` default auxiliary
  model slot.

## Bottom line
`/goal`'s "Judge API" isn't a separate product or external dependency — it's a
second, independently-configurable LLM call Hermes makes after every autonomous turn
to get a structured, unbiased verdict on whether the goal is actually done, blocked,
or should continue, so the loop doesn't have to trust the same model that just did
the work to also grade its own homework. No code was changed this session —
investigation only.

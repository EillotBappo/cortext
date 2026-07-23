# cortext

**Opus thinks, Haiku grinds.**

A Claude Code plugin that turns your expensive main agent into a *planner* and
pushes the grunt work down to cheap Haiku subagents — each handed the *minimum*
context it needs, running in its own git worktree, with a live dashboard showing
exactly where your tokens go.

- **Cheap by default.** Every subagent runs on **Haiku**. The expensive model
  only plans and integrates; it never grinds.
- **Grounded, then scoped.** The orchestrator finds facts first with the
  highest-signal tool the project has (a code-graph like **tokensave**, memory
  like **claude-mem**, or `grep`), then hands each subagent a tiny scoped brief —
  so agents *judge*, they don't *discover*.
- **Provably complete.** Subtasks declare what they must cover; `cortext-verify`
  fails the run on any gap. No silently-dropped items.
- **The spend is visible.** A live TUI, inline agent-panel rows, and a status-line
  summary show tokens and how much grunt-work context stayed off your main window.
- **Isolated.** Every `ship` subtask edits in its own worktree — parallel edits
  can't collide.

In a [live head-to-head](BENCHMARK.md), cheap Haiku subagents matched a premium
Opus run's coverage exactly, for **~4–12× less money** — once they ground with
search instead of reading whole files (the discipline the plugin now enforces).

> Inspired by [firstmate](https://github.com/kunchenguid/firstmate)'s crew model —
> pointed at a different goal: cheap over peer, scoped over full, spend you can see.

## How it works

```
  You ──▶ Orchestrator (Opus/Sonnet)
             │  ground with grep/tokensave, write a tiny brief per subtask
             ▼
        ┌────────────┬────────────┐
        ▼            ▼            ▼
   ct-haiku      ct-haiku      ct-haiku      ← cheap, scoped, isolated worktrees
   (scout)        (ship)        (ship)
        └────────────┴────────────┘
             │  terse structured results + coverage line
             ▼
        Orchestrator integrates, verifies coverage, decides next round
```

1. **Give the main agent a task with independent parts.** The bundled `delegate`
   skill auto-triggers when it splits into 2+ subtasks.
2. **Ground + decompose.** The orchestrator finds the facts (grep / tokensave /
   claude-mem), then writes a minimal brief per subtask into `.cortext/tasks.json`
   — exact paths, one success criterion, and optionally an `expects` coverage
   contract.
3. **Dispatch** each subtask to a Haiku subagent (`ct-haiku-scout` for research,
   `ct-haiku-ship` for edits), backgrounded and worktree-isolated. Prompts are
   tagged `[cortext:<tag>]`.
4. **Integrate.** For a `ship` worktree, review its uncommitted diff
   (`git -C <worktreePath> diff`) and apply it — do **not** `git merge` the
   branch (its base is a snapshot of your dirty tree).
5. **Verify.** Run `cortext-verify` — a run isn't done while any subtask is under
   its coverage contract.

> `.cortext/` holds the run manifest. Add it to `.gitignore`.

## Install

```
/plugin marketplace add EillotBappo/cortext
/plugin install cortext@cortext
```

Restart or `/reload-plugins`. Agents register at session start.

<details><summary>Running from a clone (no plugin install)</summary>

```
git clone https://github.com/EillotBappo/cortext
claude --plugin-dir ./cortext
```

Or symlink the tools onto your PATH and copy the agents into your project:

```
ln -sf "$PWD/bin/cortext-monitor" ~/.local/bin/cortext-monitor
cp agents/ct-haiku-*.md .claude/agents/   # then restart the session
```
</details>

## Use

Give Claude a task with independent parts, e.g. *"add a `deprecated` flag to every
exported function in `src/api/` and update the docs."* It engages cortext; you can
also nudge it: *"delegate this with cortext."* Watch live with `cortext-monitor` in
a second terminal.

## Watching the spend

cortext shows the spend in three places — the first two need no second window:

**Inline agent-panel rows** (`subagentStatusLine`) — model · progress bar · out:
```
⛵ t1 · haiku · ▓▓▓▓░░░░ 50% (6/12) · 1.1k out · ●
⛵ t2 · haiku · ▓▓▓▓▓▓▓▓ 100% (20/20) · 6.0k out · ✓
```

**Main status line** (`cortext-status`) — your normal line plus a run summary,
nothing when idle:
```
Opus 4.8  my-app  main   ⛵ 3/5 · 218k out · 1.2M off-ctx · t4 run
```

**Sidecar TUI** (`cortext-monitor`) — the full dashboard in a second terminal:
```
cortext monitor  (a1b2c3.jsonl)   ^C to quit

  t1     scout ▓▓▓▓▓▓▓░░░  72%  (13/18 calls)  ✓ done
         out  ▓▓░░░░░░░░ 8.2k/40.0k  ↓39.3k ctx   audit auth callers
         • Read auth.swift

  Σ 1 agents · 8.2k out · ~39k tokens ground on Haiku, off your main context
```

```
cortext-monitor            # live dashboard          cortext-monitor --once  # one frame (CI)
cortext-monitor --selftest # parser self-checks      keys: q quit · r rescan · c clear
```

<details><summary>What the numbers mean</summary>

The monitor reads only what Claude Code already writes: the per-subagent
transcripts (`~/.claude/projects/<slug>/<session>/subagents/agent-*.jsonl`) plus
`.cortext/tasks.json` for labels and budgets. No hooks, no instrumentation.

Two token figures per agent:
- **`out`** — output tokens the subagent generated. cortext's real cost signal.
- **`↓ctx`** — context pushed down on its heaviest turn (mostly cached tool
  schemas + system prompt). Large even for a tiny brief, but billed ~0.1×.

The **`Σ` footer** / `off-ctx` sums each agent's peak context — grunt-work that ran
on Haiku and never entered your Opus window. Progress % is `tool_calls / budget` —
a liveness estimate, not ground truth.
</details>

## What's in the box

```
cortext/
├── skills/delegate/SKILL.md   # the orchestration discipline (auto-triggers)
├── agents/
│   ├── ct-haiku-scout.md      # Haiku read-only researcher (search-first)
│   └── ct-haiku-ship.md       # Haiku edit worker (worktree)
├── settings.json              # registers status lines + the suggest hook
└── bin/
    ├── cortext-monitor        # sidecar TUI (pure-stdlib Python 3, no deps)
    ├── cortext-statusline     # per-subagent rows in the agent panel
    ├── cortext-status         # main status-line + live run summary
    ├── cortext-suggest        # hook: nudges toward delegation on fan-out prompts
    └── cortext-verify         # coverage gate: exits non-zero on any gap
```

## Benchmark

We tested the same code-audit several ways. Cheap Haiku agents (what cortext runs)
matched a premium Opus run's coverage — for a fraction of the cost:

```
Tokens for the same audit  (lower is better)

  cortext · Haiku    █████████████████████████·········  636k   → 76 sites ✓
  Agent tool · Opus  ██████████████████████████████████  871k   → 76 sites ✓
  grep · no agents   ▏                                    ~few k → misses judgment sites
```

Same coverage, **~7× less money** than Opus, faster. The catch: a premium model
still has real value — it reasons deeper and counts more accurately. So match the
tool to the goal:

| Goal | Best tool |
|---|---|
| Exact count of known patterns | **grep** — cheapest, exact |
| Find real change-sites cheaply | **cortext (Haiku)** — Opus coverage, ~⅓ the money |
| Plan the edit (downstream refs, dead code) | **Opus agent** — deeper reasoning, pricier |
| Determinism / structured replay | **Workflow** |

**How cortext beats a raw agent** — the benchmark exposed three failure modes; the
plugin's discipline fixes each without a bigger model:

- **Over-counting** (raw Haiku: 154 refs vs true 140) → **ground first** with
  tokensave/grep, then agents only judge the known set.
- **Dropped items** (Opus silently returned 54/55) → **the coverage gate**
  (`expects` + `cortext-verify`) keeps the run red until every item is accounted.
- **Reading whole files to hunt** (a scout burned 189k where Opus grepped for 39k)
  → **search-first scouts**: 92,901 → 22,656 tokens on a re-run, same coverage.

Full numbers, both runs, and caveats: **[BENCHMARK.md](BENCHMARK.md)**.

## Limitations

- Needs **Python 3** (standard library only).
- Progress % is a tool-call estimate, not a true step count.
- Subagent model control depends on your harness honouring `model: haiku`.
- `cortext-verify` gates on *reported* coverage — it catches under-reporting or a
  missing subagent, not one that falsely *claims* an id. Grounding `expects` from
  grep (not the agent) keeps the contract honest.

## License

MIT © EillotBappo

# cortext

**Opus thinks, Haiku grinds.**

A Claude Code plugin that turns your expensive main agent into a *planner* and
pushes the grunt work down to cheap Haiku subagents — each one handed the
*minimum* context it needs, running in its own isolated git worktree, with a
live terminal dashboard showing exactly where your tokens are going.

Most subagent tooling runs peer agents on your *own* model with *full* context
and leaves the spend invisible. cortext is built around the opposite bet — that
the cheap way is the right way when the work is grunt work:

- **Cheap by default.** Every subagent runs on **Haiku**, not your planner
  model. The expensive model only plans and integrates; it never grinds.
- **Minimal context, on purpose.** Each subagent gets a *scoped brief* — the
  exact paths and one success criterion, nothing else — dispatched through a
  4–6 tool agent type so its context isn't bloated by tool schemas it'll never
  use. Small context is the whole point, not a side effect.
- **The spend is the product.** A live token/progress TUI, inline agent-panel
  rows, and a main-status-line summary all show where tokens go and how much
  grunt-work context stayed off your main window. You never fly blind.
- **Isolated, never colliding.** Every `ship` subtask edits in its own git
  worktree, so parallel edits can't step on each other.

In a [side-by-side audit benchmark](#benchmark-cheap-agents-vs-a-premium-model),
cheap Haiku subagents found **the exact same 76 change-sites as a premium Opus
run** — using **27% fewer tokens** and finishing in **a third of the wall-clock**.
A bigger model paid ~7× more for zero extra coverage.

> Inspired by [firstmate](https://github.com/kunchenguid/firstmate)'s crew
> model — cortext takes the crew idea somewhere else: cheap over peer, scoped
> over full, and spend you can see.

## How it works

```
  You ──▶ Orchestrator (Opus/Sonnet)
             │  decompose task, write a tiny brief per subtask
             ▼
        ┌────────────┬────────────┐
        ▼            ▼            ▼
   ct-haiku      ct-haiku      ct-haiku      ← cheap, scoped, isolated worktrees
   (scout)        (ship)        (ship)
        └────────────┴────────────┘
             │  terse structured results
             ▼
        Orchestrator integrates, decides next round

   cortext-monitor  ◀── tails the session transcript + .cortext/tasks.json
   (2nd terminal)       → live per-agent progress %, token bar, current action
```

1. **You give the main agent a task.** The bundled `delegate` skill
   auto-triggers when the task splits into 2+ independent subtasks.
2. **The orchestrator decomposes** it and, for each subtask, writes a minimal
   brief — the exact paths/symbols needed, one success criterion, nothing else.
   It records the plan in `.cortext/tasks.json`.
3. **Each subtask is dispatched** to a Haiku subagent (`ct-haiku-scout` for
   read-only research, `ct-haiku-ship` for edits) in a background, worktree-
   isolated run. Every subagent prompt is tagged `[cortext:<tag>]`.
4. **The monitor** (a separate terminal) reads the transcript Claude Code already
   writes and renders live bars. No instrumentation, no hooks — it just tails
   the native output.
5. **The orchestrator reads results back.** For a `ship` worktree it reviews the
   subagent's **uncommitted** diff (`git -C <worktreePath> diff`) and applies it
   to the working tree — it does *not* `git merge` the worktree branch, whose
   base is a snapshot of your dirty tree. Then it decides what to do next.

> `.cortext/` is created in your repo to hold the run manifest. Add `.cortext/`
> to your `.gitignore` so it never lands in a commit.

### Where the numbers come from

The dashboard joins two sources:

- **`.cortext/tasks.json`** — written by the orchestrator each run. Supplies each
  agent's label, mode, tool-call budget, and token ceiling.
- **The per-subagent transcripts** that Claude Code writes to
  `~/.claude/projects/<slug>/<session-id>/subagents/agent-*.jsonl` — one file per
  subagent, the native log. For each, the monitor sums `usage.output_tokens` for
  the **out bar**, counts `tool_use` blocks for the **progress bar**, reads the
  last `tool_use` for the **current-action** line, and detects completion from
  the final `end_turn`.

Each agent shows **two** token numbers:

- **`out`** — output tokens the subagent generated. cortext's real cost signal.
- **`↓ctx`** — context pushed down on its heaviest turn (mostly cached tool
  schemas + system prompt). Large even for a tiny brief, but billed at ~0.1×.

<details><summary>Why <code>↓ctx</code> is big but not a big bill</summary>

`↓ctx` is fixed per-subagent overhead — system prompt, every tool schema,
injected `CLAUDE.md` — not your brief. Dispatching through the scoped
`ct-haiku-*` agents (4–6 tools) instead of a full-access `general-purpose` agent
(every tool schema) is the biggest lever you control. Most of `↓ctx` is
`cache_read`, billed ~0.1×. Progress % is an estimate (`tool_calls / budget`) —
a liveness signal that pins at 100% past budget rather than faking "done".
</details>

### The bottom line: tokens saved

The dashboard closes with the number that proves the whole point — a **`Σ`
footer** counting the grunt-work context that ran on Haiku and never touched
your Opus window:

```
Σ 2 agents · 11.3k out · ~93k tokens ground on Haiku, off your main context
```

That same `off-ctx` figure also rides the main status line, so you see the save
live without a second terminal.

```
cortext monitor  (a1b2c3.jsonl)   ^C to quit

  t1     scout ▓▓▓▓▓▓▓░░░  72%  (13/18 calls)  ✓ done
         out  ▓▓░░░░░░░░ 8.2k/40.0k  ↓39.3k ctx   audit auth callers
         • Read auth.swift
  t2     ship  ▓▓▓░░░░░░░  31%  (6/20 calls)   / run
         out  ▓░░░░░░░░░ 3.1k/60.0k  ↓54.6k ctx   fix null deref in cart
         • Edit cart.swift

  Σ 2 agents · 11.3k out · ~93k tokens ground on Haiku, off your main context
```

## Install

```
/plugin marketplace add EillotBappo/cortext
/plugin install cortext@cortext
```

Then restart or `/reload-plugins`. The `cortext-monitor` command is added to your
PATH while the plugin is enabled.

**Running from a clone (no plugin install):** symlink the monitor onto your PATH
once so `cortext-monitor` works from any directory:

```
ln -sf "$PWD/bin/cortext-monitor" ~/.local/bin/cortext-monitor
```

The Haiku worker agents live in `agents/`. A plugin install registers them
automatically; from a clone, copy them to `.claude/agents/` in your project (or
`~/.claude/agents/` for all projects) and **restart the session** — agents only
load at startup, so a mid-session copy won't resolve until you reload.

### Try it without installing

```
git clone https://github.com/EillotBappo/cortext
claude --plugin-dir ./cortext
```

## Use

1. Give Claude a task with independent parts, e.g.
   *"Add a `deprecated` flag to every exported function in `src/api/` and update
   the docs for each."*
2. When it engages cortext, open a second terminal in the same project and run:
   ```
   cortext-monitor
   ```
3. Watch the crew grind. The main agent integrates results as they land.

You can also nudge it explicitly: *"delegate this with cortext"*.

### Monitor commands

```
cortext-monitor            # live dashboard for the current project
cortext-monitor --once     # print one frame and exit (CI / non-TTY)
cortext-monitor path.jsonl # watch a specific transcript
cortext-monitor --selftest # run the parser self-checks
```

Live keys inside the dashboard: **`q`** quit · **`r`** rescan (re-pick the active
session) · **`c`** clear + repaint.

## What's in the box

```
cortext/
├── .claude-plugin/
│   ├── plugin.json         # plugin manifest
│   └── marketplace.json    # so the repo installs as its own marketplace
├── skills/delegate/SKILL.md   # the orchestration discipline (auto-triggers)
├── agents/
│   ├── ct-haiku-ship.md    # Haiku edit worker (worktree)
│   └── ct-haiku-scout.md   # Haiku read-only researcher
├── settings.json           # registers both status lines + the suggest hook
└── bin/
    ├── cortext-monitor     # the sidecar TUI (pure-stdlib Python 3, no deps)
    ├── cortext-statusline  # per-subagent rows (with progress bar) in the agent panel
    ├── cortext-status      # main status-line: normal line + live run summary
    ├── cortext-suggest     # UserPromptSubmit hook: nudges toward delegation
    └── cortext-verify      # deterministic coverage gate: fails a run with gaps
```

## Two views

cortext gives you the spend in two places:

**1. In Claude Code itself — no second window needed.** Two native surfaces:

- **Per-subagent rows** (`subagentStatusLine`): each subagent gets a compact
  cortext row in Claude's agent panel, now with a live **progress bar** — the
  statusline joins each row to its own transcript by agent id and reads the
  manifest budget, so you see calls-vs-budget inline:

  ```
  ⛵ t1 · haiku · ▓▓▓▓░░░░ 50% (6/12) · 1.1k out · ●
  ⛵ t2 · haiku · ▓▓▓▓▓▓▓▓ 100% (20/20) · 6.0k out · ✓
  ```

- **Main status line** (`statusLine`, `cortext-status`): your normal line
  (model · dir · branch) with a run summary appended while cortext is active,
  and nothing extra when it's idle:

  ```
  Opus 4.8  my-app  main   ⛵ 3/5 · 218k out · 1.2M off-ctx · t4 run
  ```

Between the two, everyday runs never need the sidecar.

**2. The sidecar TUI** (`cortext-monitor`) — optional second terminal with the
full-screen dashboard: per-agent bars, the `↓ctx` context column, and the live
current-action line. Reach for it when you want the rich continuous view; a full
multi-line live dashboard can't render *inside* Claude's chat pane (the harness
owns it), which is why this half stays a sidecar.

### Organic activation

A `UserPromptSubmit` hook (`cortext-suggest`) watches for fan-out-shaped prompts
— "across all…", "for each…", several file paths — and drops a one-line nudge to
consider the `delegate` skill. It's deliberately quiet: it stays silent on
single-target tasks and when you've already said "cortext".

## Benchmark: cheap agents vs. a premium model

We ran the same **read-only code-audit** four ways — grep, a structured Workflow,
and the Agent tool on both Haiku and Opus. Here's the cost of each, with the
coverage it achieved (measured, not modelled — [full table + method](BENCHMARK.md)):

```
Tokens to complete the same code-audit  (lower is better)

  cortext · 9× Haiku    █████████████████████████·········  636k   → 76 sites found
  Workflow · 9× Haiku   ███████████████████████████·······  684k   → 53 sites found
  Agent tool · 9× Opus  ██████████████████████████████████  871k   → 76 sites found

  grep · no agents: ~few k tokens, exact — but only finds the 53 patterns you name
```

**cortext (cheap Haiku agents) found the exact same 76 change-sites as the
premium Opus run** — using 27% fewer tokens (636k vs 871k) and finishing in a
third of the wall-clock (40s vs 121s). At Opus's ~5×/token price that's roughly
**~7× less money for identical coverage.** Opus even silently dropped one item
(54/55 scanned) where Haiku returned all 55.

The honest part: Opus wasn't useless — it counted references more accurately and
reasoned deeper (traced downstream consumers, flagged dead imports). Its value is
**reasoning depth, not finding more.** So match the tool to the goal:

| Goal | Best tool |
|---|---|
| Exact count of known patterns | **grep** — cheapest, exact |
| Find the real change-sites cheaply | **cortext (Haiku agents)** — Opus coverage, ~⅓ the money, fastest |
| Plan the edit (downstream refs, dead code) | **Opus agent** — deeper reasoning, but pricier |
| Determinism / structured replay | **Workflow** |

cortext owns the middle: breadth coverage at cheap-agent cost. See
[BENCHMARK.md](BENCHMARK.md) for the full numbers and caveats.

### Closing the two gaps the benchmark exposed

Plain Haiku agents matched Opus on coverage but had two weaknesses — and *so did
Opus*. cortext's discipline is built to beat both on exactly these axes, without
paying for a bigger model:

- **Over-counting** (Haiku reported 154 refs vs a true 140) → **ground first.**
  The orchestrator runs `rg`/`grep`/`ast-grep` for the exact inventory *before*
  dispatching, then hands each scout its slice to *judge*, not *discover*. Grep
  supplies the denominator; agents supply the judgment. No agent guesses a count.
- **Silently dropping items** (Opus returned 54/55) → **a coverage gate.** Each
  subtask can declare `expects` (a count or an id list) in the manifest; each
  subagent ends with a `cortext-coverage:` line; **`cortext-verify` reconciles
  the two and exits non-zero on any gap**, so a run can't be called done while an
  item is unaccounted for:

  ```
  $ cortext-verify
  cortext-verify: 1/2 subtasks fully covered  — re-dispatch the ✗ subtasks
    ✓ t1: covered 140/140
    ✗ t2: missing 1: checkout.swift
  ```

- **Noisy raw output** (Haiku's ~90 raw hits → 76 real) → the scout contract now
  splits `confirmed:` from `candidate:` and forbids padding the confirmed set
  with downstream references.

Net: same cheap-Haiku cost as the raw Agent tool, but **grounded** (exact counts)
and **provably complete** (the gate is red until every expected item is covered)
— the two things a premium model charged ~7× more for and *still* got wrong.

## Limitations

- The monitor needs **Python 3** (standard library only — nothing to install).
- Progress % is a tool-call estimate, not a true step count (see above).
- The `Σ` off-context total sums each agent's peak context (not a per-turn sum),
  so it's an honest floor for tokens kept off your main agent — not an upper bound.
- Subagent model control depends on your harness honouring `model: haiku` in the
  agent definition / Agent tool call.
- `cortext-verify` gates on *reported* coverage — it catches a subagent that
  under-reports or goes missing, but can't detect one that *claims* an id it
  didn't truly cover. Grounding the `expects` list from step-0 grep (not from the
  agent) is what keeps the contract honest.

## License

MIT © EillotBappo

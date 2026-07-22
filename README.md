# cortext

**Opus thinks, Haiku grinds.**

A Claude Code plugin that turns your expensive main agent into a *planner* and
pushes the grunt work down to cheap Haiku subagents — each one handed the
*minimum* context it needs, running in its own isolated git worktree, with a
live terminal dashboard showing exactly where your tokens are going.

Inspired by [firstmate](https://github.com/kunchenguid/firstmate)'s crew model,
but pointed at a different problem. firstmate is about **parallel isolation** —
you supervise several full peer agents in separate windows. cortext is about
**cost economy + visibility**:

| | firstmate | cortext |
|---|---|---|
| Subagent model | same as yours | **cheap Haiku** |
| Context per agent | full | **minimal scoped brief** |
| Focus | parallel PRs | **spend less, see the spend** |
| Isolation | worktrees | worktrees (kept) |
| UI | tmux windows | **token + progress TUI** |

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

### `out` vs `↓ctx` — two token numbers, both real

The monitor shows **two** token figures per agent, because they answer different
questions and differ by ~10×:

- **`out`** — cumulative **output** tokens the subagent generated. This is the
  work it produced and cortext's real cost signal (output is the expensive half).
- **`↓ctx`** — the **context pushed down** to the model on its heaviest turn
  (`input + cache_read + cache_creation`). This is what Claude Code's own agent
  panel shows as **`↓ N tokens`**. It's dominated by cache reads, so it's large
  even when the brief is tiny.

So a Haiku scout that read three files and wrote a two-line answer might show
`5.4k out · ↓39k ctx`: 5.4k of actual work, 39k of (mostly cached, cheap)
context. If those two numbers looked like they disagreed before — they weren't
wrong, they were measuring different things.

> **Why is `↓ctx` so big for a tiny brief?** It's fixed per-subagent overhead —
> the system prompt, **every tool schema**, and any injected `CLAUDE.md` — not
> your brief. The biggest lever you control: dispatch through `ct-haiku-scout` /
> `ct-haiku-ship` (4–6 tools each) rather than a full-access `general-purpose`
> agent, which loads *every* tool schema into context. Most of `↓ctx` is
> `cache_read`, billed at ~0.1×, so a big number is not a big bill.

> The progress % is an honest **estimate** — `tool_calls / budget`. It's a
> liveness signal, not a promise; it pins at 100% if an agent runs past its
> budget rather than faking "done". Completion is detected from the subagent's
> final `end_turn`.

```
cortext monitor  (a1b2c3.jsonl)   ^C to quit

  t1     scout ▓▓▓▓▓▓▓░░░  72%  (13/18 calls)  ✓ done
         out  ▓▓░░░░░░░░ 8.2k/40.0k  ↓39.3k ctx   audit auth callers
         • Read auth.swift
  t2     ship  ▓▓▓░░░░░░░  31%  (6/20 calls)   / run
         out  ▓░░░░░░░░░ 3.1k/60.0k  ↓54.6k ctx   fix null deref in cart
         • Edit cart.swift
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
    └── cortext-suggest     # UserPromptSubmit hook: nudges toward delegation
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
  Opus 4.8  my-app  main   ⛵ 3/5 · 218k out · t4 run
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

## Limitations

- The monitor needs **Python 3** (standard library only — nothing to install).
- Progress % is a tool-call estimate, not a true step count (see above).
- Subagent model control depends on your harness honouring `model: haiku` in the
  agent definition / Agent tool call.

## License

MIT © EillotBappo

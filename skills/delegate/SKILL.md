---
name: delegate
description: Use whenever a task splits into 2+ independent subtasks — even small, quick, or mechanical ones (multi-file edits, parallel research/audits, batch refactors, "do X across all Y", "check A and B and C"). Size is not the bar; breadth is. Delegates each subtask to a cheap Haiku subagent in an isolated git worktree with a minimal scoped brief, so the expensive orchestrator model only plans and integrates. Emits a live token/progress dashboard.
---

# cortext — Opus thinks, Haiku grinds

You are the **orchestrator**. Your job is to split the work, write each subagent
the *smallest brief that lets it succeed*, dispatch it to cheap Haiku, then read
the results back and decide what happens next. You do NOT do the grunt work
yourself.

Engage when the task has **2+ independent subtasks** that don't share mutable
state — including small ones. Two quick greps, three files to skim, a handful of
mechanical edits: if the pieces are independent, push them to Haiku rather than
grinding them yourself. If subtasks depend on each other's output, do the
dependency yourself or sequence them — don't fake parallelism.

## The loop

0. **Ground first — highest-signal tool before agents.** Before dispatching, find
   the facts yourself with the cheapest tool that answers:
   - **Semantic / memory tools if this project has them.** A code-graph MCP like
     **tokensave** (a `.tokensave/` dir → `tokensave_context` / `tokensave_search`)
     answers "where/what/who-calls" precisely and cheaply; **claude-mem** may
     already hold the answer from past work. Prefer these over grep for structural
     questions.
   - **Deterministic search** (`rg`/`grep`/`ast-grep`) for anything *countable or
     enumerable* — it gives an exact inventory and denominator for free.

   Then hand each subagent its slice of that inventory to *judge or verify* — do
   not make agents *discover* the set. This is why cortext beats a raw Haiku agent
   on the same task: agents left to discover over-counted (154 refs vs a true
   140), while grounding gives the exact count. Agents add judgment (which sites
   truly need changing); deterministic tools supply the ground truth.

   **Name the tool in each brief.** Tell the scout which high-signal tool this
   project has ("this repo has tokensave — use `tokensave_context` first, then
   grep") so it doesn't fall back to reading whole files. Skip grounding only when
   the task has no enumerable or locatable structure.

1. **Decompose.** Break the task into independent subtasks. For each, decide a
   **mode**:
   - `scout` — read-only. Research, audit, "find where X happens", "does Y hold".
   - `ship` — makes edits. Gets its own isolated git worktree so parallel edits
     never collide.

2. **Write the manifest.** Overwrite `.cortext/tasks.json` in the project root
   (create the dir). This is what the dashboard reads for labels and the
   progress denominators:

   ```json
   [
     { "tag": "t1", "label": "audit auth callers", "mode": "scout", "tool_budget": 12, "token_budget": 40000, "expects": 140 },
     { "tag": "t2", "label": "fix null deref in cart", "mode": "ship", "tool_budget": 20, "token_budget": 60000, "expects": ["cart.swift", "checkout.swift"] }
   ]
   ```
   `tool_budget` = your honest estimate of how many tool calls the subtask needs
   (it's the progress-bar denominator). `token_budget` = the token-bar ceiling.
   `expects` (optional but strongly recommended when you grounded in step 0) = the
   **coverage contract**: an integer minimum count, or the exact list of item ids
   the subtask must return. `cortext-verify` reconciles this against what the
   subagent reports and fails the run on any gap — this is what stops a subtask
   from silently returning 54/55 (the exact completeness failure the benchmark
   caught a premium model committing). Keep tags short and unique.

3. **Tell the user to watch (once per session).** Say:
   > Run `cortext-monitor` in a second terminal to watch progress live.

4. **Dispatch.** Spawn one subagent per subtask with the **Agent tool**:
   - `subagent_type`: `ct-haiku-ship` (for `ship`) or `ct-haiku-scout` (for `scout`).
     These restrict the toolset to a handful of tools, which is **most of the
     token win** — a subagent's context is dominated by its tool schemas, not
     your brief. If the Agent tool reports the type isn't found, the agents
     aren't registered yet (plugin not installed, or `.claude/agents/` added
     mid-session — agents only load at session start). **Reload/restart the
     session so they register, or install the plugin** — do NOT just fall back to
     the full-access `general-purpose` agent: it loads *every* tool schema into
     each subagent (tens of thousands of tokens) and defeats the point. If you
     truly can't register them, the least-bad fallback is a read-only agent with
     `model: "haiku"`, and say so — the token bar will run hot.
   - `isolation: "worktree"` for every `ship` subtask.
   - `run_in_background: true` so they run concurrently and you keep control.
   - **The prompt MUST start with `[cortext:<tag>]`** — this is how the dashboard
     attributes tokens and tool calls to the right task. Then a **minimal brief**:
     - the exact files/paths/symbols it needs (paths, not pasted contents — let it
       read what it needs) — for a scout, its slice of the step-0 inventory,
     - one crisp success criterion,
     - "return a terse structured result: what you changed / what you found. No prose."
     - **if you set `expects`**, tell it to end with a `cortext-coverage:` line —
       the count it handled, or the ids it covered (e.g. `cortext-coverage: 140`
       or `cortext-coverage: cart.swift, checkout.swift`). This is the machine-
       checkable half of the coverage contract.

   **Minimal-context rule:** never dump whole files or the whole task into a
   brief. Give the subagent the narrow slice its subtask needs and nothing else.
   That is the entire point — cheap tokens, small context.

5. **Integrate.** As each subagent returns, read its summary. For a `ship`
   worktree, the edit is left **uncommitted** on top of a base snapshot of your
   dirty tree — so **do NOT `git merge` the worktree branch**: its base carries
   your unrelated working changes, and merging pulls those instead of the edit.
   Get exactly what the subagent changed from the `worktreePath` the completion
   notification gives you:

   ```
   git -C <worktreePath> diff        # precisely the subagent's edit
   ```

   Review that diff, then apply the same change to your working tree
   (`git apply`, or just redo the edit yourself). Decide the next round or
   finish. Re-dispatch only what actually needs redoing.

6. **Reconcile coverage before declaring done.** If you set any `expects`, run
   `cortext-verify` (exits non-zero on any gap):

   ```
   cortext-verify        # ✗ any subtask that under-covered → re-dispatch just those
   ```

   Also cross-check counts against your step-0 deterministic inventory — if a
   scout reports more hits than grep found, it's over-counting (dedupe / demote
   to candidates), not finding extra. Do NOT tell the user the task is complete
   while `cortext-verify` is red. This gate is cortext's edge over the raw Agent
   tool: same cheap Haiku cost, but provably complete and grounded.

   **Sequential rounds on the same file:** a worktree's base is a snapshot taken
   at dispatch. If a *later* round edits a file an *earlier* round already
   changed in your working tree, that later worktree's base is stale and
   `git apply` (even `--3way`) will reject the diff. Either apply the small edit
   by hand, or finish/commit a round before dispatching another that touches the
   same files.

## Guardrails

- Small subtasks are fine — you do NOT need to reserve this for big jobs. As
  soon as there are 2+ independent pieces, delegating them to Haiku keeps the
  cheap work off your context, even when each piece is quick. Breadth is the
  only bar; size is not.
- The one thing not worth delegating is a genuine one-shot: a single subtask
  with nothing to run alongside it. That's just dispatch overhead for no
  parallelism — do it yourself.
- Don't give a `scout` write tools or a reason to edit.
- If a subagent returns garbage, that's usually too little context in the brief —
  add the missing slice and re-dispatch, don't widen it to the whole repo.
- The progress % is a tool-call *estimate*, not ground truth. Treat it as a
  liveness signal, not a guarantee.

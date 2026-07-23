# Orchestration benchmark — code-audit task

The same **read-only code-audit** run four ways: scan a fixed set of files, count
references, and flag the lines that need to change. Same task, same file batches,
same prompt across the agent runs — only the orchestrator / model differs.

The point isn't "cortext wins everything." It's that **cheap parallel agents get
the same coverage as a premium model for a fraction of the cost** — and that the
right tool depends on the goal.

## Results

| Metric | grep (0 agents) | Workflow (9× Haiku) | Agent tool (9× Haiku) | Agent tool (9× Opus) |
|---|---|---|---|---|
| Agents | 0 | 9 | 9 | 9 |
| Tokens used | ~few k | 683,617 | **635,730** | 871,398 |
| Tool uses | 3 | 78 | 55 | 69 |
| Wall-clock | <2s | ~84s | **~40s** | ~121s |
| Items scanned | 55 | 55 | 55 | 54 |
| Reference count (truth = 140) | 140 | 141 | 154 | 139 |
| **Real change-sites flagged** | 53 | 53 | **76** | **76** |
| Change-sites reported (raw) | 53 | 53 | ~90 | ~158 |

The **Agent tool + Haiku** column is what cortext does: cheap parallel subagents
with agent judgment.

## Findings

- **Deterministic tooling (grep) vs LLM agents.** For a pure count, grep was ~2
  orders of magnitude cheaper (~few k vs ~640–870k tokens) and exact — but it
  only finds patterns you explicitly name, and missed a whole category of
  change-sites nobody grepped for (53 vs the agents' 76).

- **Agent tool + Haiku matched Agent tool + Opus on coverage.** Both flagged the
  same **76 real change-sites** — Opus found no additional breakage. Opus cost
  **+37% tokens** (871k vs 636k), **~3× wall-clock** (121s vs 40s), and at Opus's
  ~5× per-token price, roughly **~7× more money** for identical coverage.

- **Opus reasoned deeper and counted more accurately** — nailed the reference
  count (139 vs truth 140; Haiku over-counted at 154) and traced downstream
  consumers / flagged dead imports. Its edge was reasoning depth, not finding
  more change-sites.

- **Opus was less reliable on completeness** — it silently dropped one item
  (returned 54/55) where Haiku returned all 55, and its raw change-site count was
  the noisiest (~158, inflated by counting downstream references).

- **Workflow vs raw Agent tool (both Haiku)** — near-tie on tokens (684k vs
  636k). The Workflow cost ~7% more tokens and ~2× wall-time (structured-output
  schema retries + orchestration) and, on this task, its constrained output
  surfaced fewer real sites (53 vs 76). Its upside is the schema guarantee and
  deterministic replay/journal, not raw discovery.

## Takeaway — match the tool to the goal

| Goal | Best choice |
|---|---|
| Exact count of known patterns | **grep** — cheapest, exact |
| Find real change-sites cheaply | **Agent tool + Haiku (cortext)** — same coverage as Opus, fastest, ~⅓ the money |
| Plan the actual edit (downstream refs, dead code) | **Agent tool + Opus** — richer reasoning, but pricier and dropped an item |
| Determinism / structured guarantee / replay | **Workflow orchestration** |

**Net:** on this task a bigger model paid ~7× more for zero extra coverage — its
value was reasoning depth and count accuracy, not finding more. Use deterministic
search for counts, cheap parallel agents for breadth, and premium models only
where reasoning depth actually changes the outcome. cortext is the middle column:
breadth coverage at cheap-agent cost.

---

# Second run — native subagent vs cortext (and the search-first fix)

A live head-to-head on a real repo. Task: audit `Wallet/Views/` for every SwiftUI
`View` struct declaration, with a count. Generic-aware grep ground truth = **73**
across 22 files. Native = one full-access `general-purpose` subagent on Opus.
cortext = two scoped Haiku scouts, each grounded with its file slice.

| Metric | naive grep | Native (1× Opus) | cortext v0.7 (scouts read files) | cortext v0.8 (search-first) |
|---|---|---|---|---|
| Coverage /73 | 71 ✗ | **73 ✓** | **73 ✓** | **73 ✓** |
| Tokens | ~few k | 39,065 | 189,241 | **~47k**\* |
| Tool uses | 1 | 4 | 23 | 4 |
| Wall-clock | <2s | 86s | ~31s (parallel) | **~31s** |

\* One scout group re-measured under the fix: **92,901 → 22,656 tokens (−75.6%)**,
identical 37/37 coverage. The other is projected to a similar cut.

**Findings:**

- **Both agent runs beat naive grep.** They found two generic
  `struct FilterSection<Content: View>: View` declarations a naive regex misses.
  Agent judgment > naive pattern.
- **Out of the box, cortext LOST on tokens** (189k vs 39k). Its scouts *read whole
  files* to enumerate (23 tool calls); the single Opus agent *grepped* (4 calls).
  Same coverage, ~5× the tokens — the scouts' **method** was the defect, not the
  model.
- **The fix (v0.8): scouts ground with search first** (grep/rg, or a semantic tool
  like tokensave when the project has one), and Read only to verify a hit.
  Re-running the worst scout dropped it 75.6% with identical coverage — which
  flips the result: search-first cortext beats the native Opus subagent on money
  (~4–12×, per the 5–15× per-token price ratio) and wall-clock, and ties coverage.

**Lesson:** cortext's edge over a native subagent isn't the cheap model alone —
it's the *discipline* (search-first grounding + the coverage gate). When scouts
abandon it and read files, a premium single agent wins. v0.8 enforces search-first
in the scout contract.

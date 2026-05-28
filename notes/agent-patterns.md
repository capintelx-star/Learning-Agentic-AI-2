# Agent Design Patterns — Day 4 Notes

Reference for the five canonical patterns (from Anthropic's *Building effective agents*),
plus when *not* to use each and a concrete fit from my own projects. Every agent built in
Weeks 3 and 8–11 will be one (or a hybrid) of these.

## The frame that matters most

**Workflow vs. agent.** Four of these five are *workflows*: the control flow lives in my
code, the LLM fills in the steps. Only a true *agent* lets the model drive its own loop and
decide when it's done.

The most expensive default-mistake at my level is reaching for an autonomous multi-agent
system when a deterministic workflow would be more reliable, cheaper, and debuggable.
**Bias: start with the simplest thing; add orchestration only when it *measurably* improves
outcomes.**

Two things the patterns don't say out loud:
- They **compose**. Real systems are hybrids — a router at the front, orchestrator-workers
  in one branch, evaluator-optimizer wrapping the output. Ask "which pattern fits each
  *seam*," not "which pattern is my system."
- The hard part is always the **contract** between steps (the data structure passed, what a
  worker guarantees to return, what the evaluator's criteria literally are). The pattern is
  trivial; the interface is where it breaks.

---

## 1. Prompt chaining

Decompose into fixed sequential steps; each step's output feeds the next, optionally with a
gate between steps that can bail early.

- **Solves:** tasks where one-shotting tanks quality. Trade latency for accuracy by giving
  the model one job per step.
- **Don't use when:** steps don't actually depend on each other (parallelize), or a single
  well-structured prompt already nails it. Chaining for ceremony adds latency and failure
  points.
- **Real example:** outline → gate check → write sections; or generate copy → translate.
- **My fit:** CapIntelX content engine — angle/outline → gate ("does this outline carry the
  thesis?") → draft → tighten to voice. The gate is what makes it more than a long prompt.

## 2. Routing

Classify the input, dispatch to a specialized handler.

- **Solves:** distinct input categories that each want different handling (or different cost
  tiers).
- **Don't use when:** categories blur, or classification is as hard as the task itself —
  then I've just added a misroute failure mode. Skip if one general prompt is acceptable.
- **Real example:** support triage (billing/technical/general); cost routing — cheap model
  for easy queries, frontier model for hard ones.
- **My fit:** learning app — route a student turn to {explain, generate practice, grade,
  hint}. Or cost-routing inside the bots so trivial checks don't hit the top model.

## 3. Parallelization

Run subtasks concurrently. *Sectioning* = independent chunks. *Voting* = same task N times
for consensus/confidence.

- **Solves:** latency (sectioning) or variance reduction / confidence (voting).
- **Don't use when:** subtasks are dependent (that's a chain), or aggregation cost exceeds
  the benefit. Voting is N× spend — only when stakes justify it.
- **Real example:** one model generates while another screens for guardrails in parallel;
  run a vuln check 3× and flag if any trip.
- **My fit:** Polymarket — score a market's edge across independent signals (base rate, news
  sentiment, liquidity) in parallel, then combine.

## 4. Orchestrator–workers

A central LLM *dynamically* decomposes the task, delegates to workers, synthesizes results.

- **Solves:** problems where the subtasks can't be predefined — decomposition depends on the
  input.
- **Don't use when:** I *can* predefine the subtasks. Then it's just parallelization:
  cheaper and more predictable. Dynamic decomposition is the only thing I'm paying extra
  for — don't buy it if I don't need it.
- **Real example:** deep-research clones; multi-file code changes where the orchestrator
  decides which files to touch.
- **My fit:** Polymarket multi-agent researcher (capstone candidate / Day 11 build) —
  orchestrator looks at a specific market, decides which research threads matter for *that*
  one, spawns workers.

## 5. Evaluator–optimizer

Generator produces, separate evaluator critiques against criteria, generator refines, loop
until criteria met or max iterations.

- **Solves:** tasks with articulable criteria where iteration genuinely improves output —
  like a human reviewer doing rounds.
- **Don't use when:** I can't write good criteria, the first output is usually fine (wasted
  loops), or the evaluator can't judge better than the generator. **Best version uses an
  *objective* evaluator** (tests pass / schema validates), not vibes — that's where it's
  clearly worth the calls.
- **Real example:** code that must pass a test suite (evaluator = test runner); translation
  refinement.
- **My fit:** content engine — draft against a CapIntelX voice rubric, critic scores
  hook/clarity/voice, refine. Or Polymarket — generate a trade thesis, critic red-teams the
  bear case, refine the sizing.

---

## Selection cheat-sheet

| If I'm tempted to... | First ask | Often the answer is |
|---|---|---|
| Build a multi-agent system | Can I predefine the subtasks? | Parallelization, not orchestrator-workers |
| Add an evaluator loop | Can I write an *objective* check? | If no, probably skip it |
| Route inputs | Is classification easier than the task? | If no, just use one general prompt |
| Chain steps | Do steps actually depend on each other? | If no, parallelize |
| Reach for an "agent" | Would a fixed workflow do it? | Use the workflow |

**Default order of escalation:** single prompt → chain → route/parallelize → orchestrator-workers / evaluator-optimizer → autonomous agent. Stop at the first one that hits the bar.

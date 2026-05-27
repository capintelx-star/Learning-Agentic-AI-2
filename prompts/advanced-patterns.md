# Advanced Prompting Patterns — Personal Reference

## When to reach for these

Most prompts don't need any of this. A one-liner gets a one-liner; default Claude Code handles routine edits fine. These patterns earn their token cost on **multi-file refactors, multi-step reasoning, irreversible changes, and anywhere "the model hedged" is the failure mode**. Rule of thumb: if I'd be annoyed to throw away the output and start over, the prompt deserves structure.

---

## Pattern catalog

### 1. Role-tagged multi-file context

**Problem it solves:** Most multi-file prompts dump files into context with no signal about what each file is *for*, so Claude treats them all as equally edit-eligible and equally style-authoritative.

**When to use:**
- Refactor touches 3+ files with different roles (files-to-edit vs files-to-respect-style-of vs read-only references).
- Codebase has strong local conventions that other files demonstrate.
- I care which files Claude pattern-matches against.

**When NOT to use:**
- Single-file edits.
- Greenfield code with no existing conventions to respect.

**Skeleton:**
```
<context_files priority="foundational">
These define constraints you MUST respect. Read them first.

- path/to/file.py — one-sentence role. Note any specific gotcha.
- path/to/other.py — its role. What about it matters.
</context_files>

<context_files priority="conventions">
Read for style/patterns, do not edit.

- path/to/example.py — what convention this demonstrates.
- tests/test_x.py — the test style to mirror.
</context_files>
```

**Real example from RoomAgent refactor:**
```
<context_files priority="foundational">
- src/llm.py — the file being refactored. Note: _client is module-level,
  parse_message() is sync, _last_llm_status is a module global mutated
  inside parse_message.
- src/handlers/supplies.py — the only caller of parse_message() in the
  hot path (free_text_handler). The async def handler that today blocks
  the event loop.
- src/handlers/health.py — reads _last_llm_status. Any change to how
  that global is written must not break this reader.
</context_files>
```
Made Claude correctly identify health.py as a *reader to not break* rather than a *file to edit* — preempted an unnecessary edit pass.

---

### 2. `<not_in_context>` defensive block

**Problem it solves:** Claude fills gaps with plausible-sounding fabrications about anything not in the window — SDK behavior, deployment environment, performance characteristics. These often look like fact in the response.

**When to use:**
- Prompt involves library/SDK behavior I haven't pasted source for.
- Any claim about latency, scale, deployment topology matters to the design.
- Working on a single piece of a larger system Claude can't see.

**When NOT to use:**
- Self-contained tasks where there's no external context to assume.
- When I genuinely want Claude to use general knowledge (e.g. "is X a common Python idiom").

**Skeleton:**
```
<not_in_context>
I have NOT given you:
- [external library source]. Assume [specific signature/behavior].
- [performance data]. Assume [concrete number range].
- [deployment topology]. State of the world: [the relevant facts].
</not_in_context>
```

**Real example from RoomAgent refactor:**
```
<not_in_context>
I have NOT given you:
- The full Anthropic SDK source. Assume anthropic.Anthropic() and
  anthropic.AsyncAnthropic() both exist with identical messages.create
  signatures (one sync, one async).
- Any benchmark of current latency. Assume parse_message() takes
  400-1500ms in practice.
- The deployment environment (it's a single-process bot, single event
  loop, no multi-worker setup).
</not_in_context>
```
Prevented Claude from inventing scenarios about multi-worker deployments or thread pools that would have polluted the concurrency analysis.

---

### 3. `<decisions_required>` forcing commitment

**Problem it solves:** Default Claude output hedges on every architectural decision — "you could go either way," "it depends on your priorities." Useless for actually shipping.

**When to use:**
- Design tasks with real trade-offs where I need a peer's opinion, not a survey.
- When I've noticed Claude defaulting to "here are the considerations" instead of picking.
- When I want defended positions I can argue against.

**When NOT to use:**
- Genuinely open-ended brainstorming (where hedging is correct).
- When the decision really does belong to me and I just want options laid out.

**Skeleton:**
```
<decisions_required>
You must take a position on each. "It depends" is not acceptable —
pick one and defend it in 1-2 sentences.

1. [Decision A or B]?
2. [Decision X, Y, or Z]?
3. [Yes/no question with real trade-offs]?
4. [Approach 1 vs approach 2]?
</decisions_required>
```

**Real example from RoomAgent refactor:**
```
<decisions_required>
1. asyncio.to_thread wrapping the sync client, OR switch to
   anthropic.AsyncAnthropic?
2. Does _last_llm_status need a lock, an asyncio.Lock, or nothing?
   Justify against the specific concurrency model.
3. Does parse_message become async, or stay sync and get awaited via
   to_thread at the call site? (These are different — pick one.)
4. What's the test strategy? Existing tests don't call the API. Do
   new tests need to? If yes, how do you mock the async path?
</decisions_required>
```
Each decision came back with a one-sentence defense referencing the actual concurrency model, not a "here are the trade-offs" survey.

---

### 4. Tool-use schema as structured-output forcing function

**Problem it solves:** Free-form responses bury the structure I actually need (rationale per change, verification steps, rollback). Pure "respond in JSON" produces JSON but Claude wriggles around required fields.

**When to use:**
- I want every item in a list to carry the same metadata (rationale, risk, etc.).
- Plans where rollback/verification matters and is easy to forget.
- Anywhere I'd reach for a JSON schema but want the prose-readable version.

**When NOT to use:**
- Conversational responses or single-shot answers.
- When the structure is obvious from context (Claude will produce it anyway).

**Skeleton:**
```
Use this tool to structure your output. Call it once with the complete
[plan/refactor/analysis]:

<tool name="apply_[action]">
  <param name="changes" type="list">
    Each item is a dict with:
      - file: path
      - operation: "[type1]" | "[type2]"
      - rationale: 1 sentence on why this specific change
      - before: the exact lines being replaced (or null if pure addition)
      - after: the exact lines after the change
  </param>
  <param name="verification_steps" type="list">
    Ordered list of commands I should run to verify it works.
  </param>
  <param name="rollback_plan" type="string">
    One sentence: how to revert if something breaks in production.
  </param>
</tool>

Then actually make the edits. The "tool call" above is the structured
plan; the file edits are the execution. Both.
```

**Real example from RoomAgent refactor:** Used exactly the skeleton above for the async client refactor. Output included a `verification_steps` list (`pytest -v`, an `inspect.iscoroutinefunction` smoke test) and a one-line `rollback_plan` (`git revert HEAD` or `git checkout --`). I wouldn't have asked for either; both turned out useful — the smoke test caught the stale `.pyc` false alarm.

---

### 5. Extended thinking scaffold

**Problem it solves:** On non-trivial multi-file edits, Claude often starts writing code before identifying the 2-3 specific traps that will bite. The fix is to name the traps in the prompt so they're thought through first.

**When to use:**
- Refactor with non-obvious second-order effects (e.g. "if X changes, does Y still hold?").
- Concurrency, ordering, or invariant-preservation work.
- Anything where I can predict the failure modes but want them resolved before editing.

**When NOT to use:**
- Simple, well-scoped edits.
- When I don't actually know what the traps are (then I'm guessing, and the scaffold misleads).

**Skeleton:**
```
<extended_thinking>
Before writing any code, think through:
1. [Specific trap #1 — phrase it as a question].
2. [Specific trap #2 — name the file/function that might break].
3. [Specific trap #3 — invariant that must be preserved].
4. [Specific trap #4 — test or verification concern].
</extended_thinking>
```

**Real example from RoomAgent refactor:**
```
<extended_thinking>
Before writing any code, think through:
1. The exact import changes needed in llm.py and test_intent_parsing.py.
2. Whether _make_response in tests needs to change shape (it doesn't —
   the response object structure is identical between sync and async
   clients, only the call is awaited).
3. Whether health.py reads _last_llm_status in a way that breaks if
   it's mutated from a coroutine vs the main thread.
4. The exact form of the concurrency test: how to construct two
   AsyncMock return values, how to use side_effect to return different
   values per call.
</extended_thinking>
```
Claude's pre-edit reasoning correctly concluded existing tests needed *zero* changes (they validate the Pydantic model, never call parse_message) — saved a wasted edit pass on a file that didn't need touching.

---

### 6. Prefilling Claude's response

**Problem it solves:** Preamble. "Great question! Here's my analysis…" Wasted tokens, slower reads, hedge-flavored openings even when I forced commitment elsewhere.

**When to use:**
- Whenever the output format is locked and I want zero preamble.
- When I want Claude to commit to a position from word one (start of response = strongest commitment).
- When parsing the response programmatically.

**When NOT to use:**
- Conversational replies.
- When the preamble might actually contain useful reasoning.

**Skeleton:**
```
<prefill>
[first heading or first words of the response]
</prefill>
```

**Real example from RoomAgent refactor:**
```
<prefill>
## Decision summary
1. **
</prefill>
```
Claude opened directly at "1. **AsyncAnthropic** — …" with the bolded choice. No "Great question, here's my plan" preamble. Forced commitment from the first word.

---

## Anti-patterns I noticed in default Claude Code usage

- **Default plans hedge on every decision.** "You could use to_thread OR AsyncAnthropic, both have trade-offs." Fix: `<decisions_required>` with "it depends is not acceptable."
- **Default file context is unordered.** Foundational files get pasted after example files; Claude attends more to recency. Fix: role-tagged context with foundational *last* in the read-this-first block.
- **Default prompts trust Claude to bound the work.** Result: Claude "improves" beyond scope — adds timeouts, retries, logging I didn't ask for. Fix: explicit `Do NOT change:` and `Out of scope:` lists.
- **Default prompts let Claude pick its own reading order.** On multi-file tasks this means edits sometimes happen before all relevant files are read. Fix: numbered read order in the prompt.
- **Default prompts don't ask for verification or rollback.** Both are cheap to produce, valuable to have. Fix: tool-use-as-structure with required `verification_steps` and `rollback_plan` fields.

---

## Decision checklist before writing the prompt

For any non-trivial prompt (multi-file edit, design decision, irreversible action):

- [ ] Have I named **each file's role** (edit / convention-source / read-only reference)?
- [ ] Is there a **`<not_in_context>`** block listing assumptions Claude shouldn't fabricate?
- [ ] Have I listed the **specific decisions** I want committed positions on, with "it depends is not acceptable"?
- [ ] Did I name the **specific traps** to think through *before* editing (extended thinking scaffold)?
- [ ] Is the **output format locked** — XML structure, tool-use schema, or explicit headers?
- [ ] Have I **prefilled** to kill preamble and force immediate commitment?
- [ ] Is there an explicit **`Do NOT change`** / **`Out of scope`** list preempting scope creep?

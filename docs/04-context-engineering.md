# Stage 4 — Context engineering: the part that actually decides quality

**Milestone:** accurate token accounting, a measured cache hit rate above 80% on a
multi-turn session, and a compaction step that lets a session run past the context
window without losing the thread.

**Time:** two sessions. This is the stage that separates people who have built an agent
from people who have used one, and it's where interview questions get specific.

---

## 4.0 The framing

Everything the model knows on turn N is in the request you sent for turn N. So the
agent's intelligence is bounded by a single question: **what is in the window, and what
did it cost to put there?**

There are only four moves:

| Move | What it means | Where it shows up |
|---|---|---|
| **Write less** | Don't put it in at all | Tool output caps, targeted reads over full reads |
| **Prune** | Remove what's no longer needed | Context editing, dropping old tool results |
| **Summarise** | Replace many tokens with fewer | Compaction |
| **Offload** | Move it out, keep a pointer | Files, memory, subagent reports |

And one orthogonal concern: **caching**, which doesn't reduce what's in the window but
changes what it costs by roughly 10x. Do caching first, because it's pure win and
requires no judgement calls.

The failure mode you're designing against is not "hit the limit and crash." It's the
slow degradation before that: a window full of stale tool output, three copies of the
same file at different revisions, and the original goal buried 200,000 tokens back.
The model gets confused, and it looks like the model got worse.

---

## 4.1 Token accounting

Two sources of truth. Use both.

**Before the request:** `POST /v1/messages/count_tokens` — same shape as a messages
request (model, system, tools, messages), returns `input_tokens`. Cheap, exact,
model-specific.

**Never use `tiktoken`.** It's OpenAI's tokenizer. It undercounts Claude by 15-20% on
prose and far more on code. Every "approximately 4 characters per token" heuristic is
wrong for the same reason. Also note that tokenizers differ *between Claude models* —
Opus 4.7 introduced a new one, so counts measured on 4.6 don't transfer. Re-baseline
per model; don't apply a multiplier.

**After the request:** `response.usage`. The trap here catches almost everyone:

```
input_tokens                 = tokens processed at FULL price (the uncached remainder)
cache_creation_input_tokens  = tokens written to cache this request
cache_read_input_tokens      = tokens served from cache this request

total prompt size = input_tokens + cache_creation_input_tokens + cache_read_input_tokens
```

`input_tokens` is **not** your prompt size. If you log only `input_tokens` and it reads
4,000 after an hour-long session, you haven't discovered a miracle — you've discovered
that 380,000 tokens came from cache. Sum all three. Build this into your logger now,
because every conclusion you draw for the rest of this stage depends on it.

**What to instrument, per turn:** all three token fields, cumulative total, percentage
of the context window consumed, tool names, wall-clock, and cost. Cost is worth
computing live; it changes your design instincts fast.

---

## 4.2 Prompt caching

### The one invariant

**Caching is a prefix match. Any byte change anywhere in the prefix invalidates
everything after it.**

The cache key is derived from the exact bytes of the rendered prompt up to each
breakpoint. Render order is fixed:

```
tools  ->  system  ->  messages
```

So a breakpoint on the last `system` block caches tools *and* system together. A change
to a tool definition invalidates everything, because tools render first. Everything
about cache design follows mechanically from those two facts.

### Design consequence: sort by stability

Classify every input by how often it changes, and make the physical order in the prompt
match the stability order:

| Changes | Where it goes |
|---|---|
| Never | Front. Tools, frozen system prompt |
| Per session | After the global prefix |
| Per turn | In `messages`, at the end |
| Per request (timestamps, UUIDs) | Eliminate, or put after the last breakpoint |

If you interpolate `Current date: 2026-07-29T14:03:22Z` into your system prompt header,
**nothing after it ever caches**, no matter how many `cache_control` markers you add.
This is the number one cause of "caching doesn't work for me."

### Mechanics

```
cache_control: { type: "ephemeral" }              // 5-minute TTL
cache_control: { type: "ephemeral", ttl: "1h" }   // 1-hour TTL
```

- Max **4 breakpoints** per request.
- Goes on a content block: a system text block, a tool definition, or a message content
  block.
- Top-level `cache_control` on the request auto-places on the last cacheable block.
  Simplest option; use it before you reach for manual placement.
- **Minimum cacheable prefix is model-dependent and not monotonic.** 512 tokens on
  Opus 5, 1024 on Opus 4.8 and Sonnet, 4096 on Opus 4.6 and Haiku 4.5. Below the
  minimum it silently doesn't cache — no error, `cache_creation_input_tokens: 0`.

**Placement patterns:**

- *Large shared system prompt:* one breakpoint on the last system block. Done.
- *Multi-turn conversation:* breakpoint on the last content block of the most recent
  turn. Earlier breakpoints stay valid as read points, so hits accrue as the
  conversation grows. This is what your agent loop wants.
- *Shared prefix, varying question:* breakpoint at the end of the **shared** part, not
  at the end of the prompt. Marking the end means every request writes a unique entry
  and nothing is ever read — a common and expensive mistake.
- *Prefix differs every request:* don't cache. You'd pay the write premium for zero reads.

### Economics

Reads cost ~0.1x base input price. Writes cost 1.25x (5-minute TTL) or 2x (1-hour).
Break-even:

- 5-minute TTL: **two** requests (1.25 + 0.1 = 1.35 versus 2.0 uncached).
- 1-hour TTL: **three** requests (2.0 + 0.2 = 2.2 versus 3.0 uncached).

So the 1-hour TTL is for bursty traffic with gaps longer than five minutes, not a
default. In an agent loop, turns arrive seconds apart, so 5-minute TTL and the
conversation keeps itself warm.

### The invalidation hierarchy

Not every change nukes everything. Three tiers, and a change only invalidates its own
tier and below:

| You change | tools cache | system cache | messages cache |
|---|:--:|:--:|:--:|
| Tool definitions (add/remove/reorder) | gone | gone | gone |
| Model | gone | gone | gone |
| System prompt content | kept | gone | gone |
| `tool_choice`, adding an image, toggling thinking | kept | kept | gone |
| Message content (normal turn) | kept | kept | gone |

Two useful implications:

- You can flip `tool_choice` per request for free. Don't over-worry about it.
- **Changing the tool set mid-conversation is the expensive operation.** So "modes" that
  swap tools are architecturally costly. The right answers are progressive disclosure
  (stage 7), or the newer mid-conversation tool-change mechanism which appends
  `tool_addition`/`tool_removal` blocks to `messages` instead of editing `tools`.

### The cache-preserving operator channel

Related trick, and a good one to know: on Opus 5 and 4.8 you can append a
`{ role: "system", content: "..." }` message *inside* `messages` rather than editing the
top-level `system` field. It carries operator authority, and because it sits after the
cached history, the prefix survives. Use it for anything your app learns mid-session:
a mode toggle, files changed on disk, remaining budget.

There's a security dimension too. The old workaround was to smuggle operator
instructions into a user-turn text block (`<system-reminder>`-style). That works, but
anything that can write to user-visible input can forge it. A real `role: "system"`
message can't be spoofed by tool output. Keep that distinction in mind for stage 8.

### Verifying

The only thing that matters: `cache_read_input_tokens` on turn 2+. If it's zero across
repeated requests with what you believe is an identical prefix, you have a silent
invalidator. Debug by diffing the rendered request bytes between two turns. The usual
suspects:

| Pattern | Why it breaks |
|---|---|
| `Date.now()` / `new Date()` in the system prompt | Prefix differs every request |
| A UUID or request id early in content | Same |
| `JSON.stringify` over an object with non-deterministic key order, or iterating a `Set` | Serialisation differs |
| Session id or user name interpolated into the system prompt | Per-user prefix, no sharing |
| Conditional system sections (`if (flag) system += ...`) | Every flag combination is a distinct prefix |
| Tool list built per user | Tools render at position 0, so nothing caches at all |

### Two operational gotchas

- **20-block lookback.** A breakpoint searches backwards at most 20 content blocks for a
  prior cache entry. Agentic turns with many tool_use/tool_result pairs blow through
  that, and the next turn silently misses. Fix: place an intermediate breakpoint every
  ~15 blocks in long turns.
- **Concurrent requests can't share a write.** A cache entry becomes readable only once
  the first response *starts streaming*. Fire N identical-prefix requests in parallel
  and all N pay full price. For fan-out: send one, await the first streamed token, then
  fire the rest.

### Pre-warming

`max_tokens: 0` runs prefill, writes the cache, and returns immediately with empty
content. Useful at process startup so the first real request isn't cold. Only worth it
when first-request latency is user-visible *and* there's a quiet moment before traffic;
with continuous traffic the first real request warms it anyway and a separate warm call
is a wasted write. Put the breakpoint on the block you share with the real request (the
system prompt), not on the placeholder user message.

---

## 4.3 Pruning and summarising

Caching bought you cost. Neither cache nor cost fixes a window that's *full*. Two
mechanisms, and they are not the same thing:

**Context editing = pruning.** Old tool results and thinking blocks are *cleared* from
the transcript. The structure survives, the content goes. Server-side via the
`context_management.edits` parameter (`clear_tool_uses_20250919`,
`clear_thinking_20251015`), or you can implement it yourself.

**Compaction = summarising.** Earlier history is replaced by a generated summary. There's
a server-side version (`compact_20260112`), and there's the version you'll build.

**Critical detail if you use server-side compaction:** append the full
`response.content` back into `messages`, not just the text. Compaction returns a
`compaction` block, and the API uses it to reconstitute the compacted history on the
next request. Extract only the text and you silently lose the compaction state — the
kind of bug that presents as "it forgot everything."

### Rolling your own compaction

Do this yourself at least once; it's where the real thinking is. The shape:

```
when total tokens > threshold (start at 70% of the window):
    keep verbatim:  system, tools, the original goal, the last K turns
    summarise:      everything in between, via a separate model call
    replace:        that middle span with one user/assistant exchange carrying the summary
```

Design questions you must answer, and there are no free answers:

- **What survives verbatim?** The original user request, always — losing the goal is the
  worst outcome. Recent turns, because that's the working state. Beyond that: the set of
  files touched and their current state is usually worth more than the transcript of how
  they were touched.
- **What does the summary need to contain?** Not a prose recap. It needs: the goal,
  decisions made and why, what's been verified versus assumed, current file state, and
  the immediate next step. Write that as an explicit instruction to the summarising
  call. Prompt it for structure, not narrative.
- **You cannot naively slice the array.** Cutting between a `tool_use` and its
  `tool_result` produces an invalid request (invariant 2 from stage 1). Thinking blocks
  carry signatures and can't be edited. So compaction must operate on *complete
  exchanges*, and it must know the block-level structure. This constraint is the reason
  compaction is fiddly rather than trivial.
- **Compaction invalidates your cache** for everything after the rewritten span. That's
  unavoidable, and it makes compaction expensive twice over. Prefer to compact rarely
  and substantially rather than often and a little.
- **Do it before you're desperate.** At 95% you may not have room for the summarising
  call's own output.

### Offloading: the underrated move

The cheapest token is the one that never enters the window. Instead of holding a
20,000-token analysis in context, have the agent write it to
`.miniclaw/notes/analysis.md` and keep a one-line pointer. The information is still
available — via `read` — but it costs nothing until needed.

This generalises: **the filesystem is your extended context.** Notes, plans, todo lists,
intermediate results. It's also why the "give the agent a scratchpad file" trick works so
well on long tasks, and it's the mechanism behind memory (stage 6) and subagent reports
(stage 5).

---

## 4.4 Build it

1. **Instrumentation first.** A per-turn log line with all three token fields, the
   correct sum, percent-of-window, and cost. Every subsequent step is measured against
   this.
2. **Add caching.** Top-level `cache_control` to start. Run a 10-turn session and record
   `cache_read_input_tokens` per turn.
3. **Do the experiment in 4.5 #1.** This is the most valuable 20 minutes in the
   curriculum.
4. **Move to explicit breakpoints:** one on the last tool definition or system block, one
   on the latest turn. Compare hit rates against the automatic version.
5. **Token accounting via `count_tokens`** before each request, so you know your budget
   before you spend it rather than after.
6. **Compaction.** Threshold at 70%. Summarise via a separate call. Handle the
   complete-exchange constraint. Log before/after token counts.
7. **Offloading.** Give the agent a `notes` directory and mention it in the system
   prompt. Watch whether it uses it; if not, be more prescriptive.

---

## 4.5 Break it on purpose (this stage's experiments are the point)

1. **The timestamp experiment.** Put `Current time: ${new Date().toISOString()}` at the
   top of your system prompt. Run a 10-turn session. Record `cache_read_input_tokens`
   per turn (it will be 0) and total cost. Now move the timestamp into the first user
   message and re-run the identical session. Compare hit rate and cost.
   **Write both numbers in NOTES.md.** This is a story you will tell in an interview,
   and having the actual numbers is the difference between sounding informed and
   sounding read-up.

2. **Add a tool mid-conversation.** At turn 5, append a new tool to the `tools` array.
   Watch `cache_read_input_tokens` go to zero and `cache_creation_input_tokens` spike to
   the full history. Now you understand why tool sets are architecturally frozen.

3. **Mark the end of the prompt instead of the end of the shared prefix.** Run 10
   requests with a large shared preamble and varying questions. Observe every request
   writing a fresh entry and reading nothing. This is the expensive mistake.

4. **Slice the message array naively.** Drop the oldest 10 messages regardless of block
   boundaries. Read the 400 about an orphaned `tool_use`. Then try dropping only the
   text and keeping the structure, and notice the thinking-signature problem.

5. **Fill the window.** Point the agent at a large repo and ask it to read everything.
   Watch it approach the limit. Note the *quality* degradation before the hard failure:
   repeated reads of the same file, forgetting the original question, contradicting
   earlier conclusions. That degradation curve is the thing compaction exists to fix,
   and seeing it is more instructive than reading about it.

6. **Compact too late.** Set the threshold to 97% and let it trigger. Note what happens
   when the summarising call has no room.

7. **Log only `input_tokens`.** Run a long cached session and watch it read absurdly
   low. Then add the other two fields. This is the accounting trap, felt rather than
   read.

---

## 4.6 Checkpoint

- [ ] Your log shows all three token fields and a correct running total.
- [ ] `cache_read_input_tokens` is non-zero from turn 2 onward, and you can state your
      measured hit rate.
- [ ] You have the timestamp experiment's before/after numbers written down.
- [ ] A session can exceed the context window and keep working, with the original goal
      still intact after compaction.
- [ ] Compaction never produces an invalid message array.
- [ ] You can name the four moves for managing context, and give an example of each from
      your own code.

**Questions to be able to answer without looking:**

- Why doesn't caching work when there's a timestamp in the system prompt? Be precise
  about the mechanism.
- What's the difference between `input_tokens` and prompt size?
- What's the difference between context editing and compaction?
- Which changes invalidate the tools cache, and which don't?
- Why can't you just drop the oldest N messages when the window fills?
- Your cache hit rate is 0 and you're sure the prefix is stable. Name five things to check.
- What's the break-even request count for the 1-hour TTL, and why is it different from 5m?

Next: [Stage 5 — Subagents](05-subagents-and-orchestration.md).

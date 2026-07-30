# Stage 5 — Subagents: context isolation, not parallelism

**Milestone:** a `task` tool that spawns a nested agent with its own message array and a
restricted toolset, and returns a structured report to the parent.

**Time:** half a session. Small in code, and the reasoning about *when* to use it is
worth more than the implementation.

---

## 5.1 What a subagent actually buys you

Most people say "parallelism." That's a side effect. The real reason is **context
isolation**.

Consider: "find every place we handle authentication and tell me if the token refresh
logic is consistent." Doing this in the main loop means reading fifteen files, and all
fifteen stay in your window for the rest of the session, crowding out the actual work.
The answer you needed was three sentences.

A subagent burns its own window doing the exploration and returns those three sentences.
The parent's window grows by three sentences instead of 40,000 tokens. That's the trade,
and it's an instance of the "offload" move from stage 4.

Parallelism is the second benefit: N independent subagents can run concurrently, and
wall-clock drops. But if you only wanted parallelism you'd parallelise tool calls, which
you already did in stage 2 for free.

### The costs, which are real

- **No shared context.** The subagent knows nothing the parent knows. Every fact it needs
  must be in the brief. This is the #1 source of bad subagent output: an
  under-specified brief producing confident, wrong work.
- **Re-establishment cost.** It re-reads files, re-derives conclusions, pays its own
  cache-cold prefix. A subagent for something you could do in four tool calls is
  strictly worse than doing it.
- **A second summarisation.** The subagent compresses to a report; the parent reads the
  report. Information is lost twice. For tasks where detail matters, that's fatal.
- **Coordination overhead.** Parallel subagents writing to the same files conflict.
  Either give them non-overlapping scopes or make them read-only.

### When to spawn, and when not to

| Spawn | Don't spawn |
|---|---|
| Wide read-only investigation across many files | Anything you could finish in a handful of tool calls |
| Genuinely independent tracks (unrelated modules) | Verification of your own work (do that in the main loop) |
| A task that will produce a lot of intermediate output you don't need | Splitting one modest job into pieces |
| A task needing a different toolset or narrower permissions | Anything where the parent needs the full detail, not a summary |

Note the second row on the right: **verification belongs in the main loop.** It's a
common instinct to spawn a "reviewer" subagent, and it usually costs more than it catches
because the reviewer lacks the context that would let it judge.

Model behaviour varies here and it's worth knowing: some Claude versions under-reach for
delegation and need encouragement, others over-reach and need an explicit cap. If your
agent is spawning five subagents for a two-file question, put a hard ceiling on spawn
count in your harness and say so in the system prompt. Prefer mechanism to instruction,
as always — but here you want both.

---

## 5.2 The design

A subagent is not a new abstraction. It's your existing loop, called recursively, with
different arguments:

```
task(description, prompt) -> report

  build a fresh ctx:  own message array, own token budget, own turn limit,
                      restricted registry (typically read-only tools),
                      inherited abort signal, inherited session log
  run the same loop
  return the final text as the tool_result
```

The interesting decisions are all at the boundary:

**Going down:** the brief. It must be self-contained. The parent's system prompt, goal,
and accumulated knowledge do not transfer. Write the brief as if handing the task to a
new hire who has repo access and nothing else. If your `task` tool's schema has one
field called `prompt`, the model will under-specify it; give it two fields
(`description` for a short label, `prompt` for the full brief) and say in the
description that the prompt must be complete and standalone.

**Coming back:** the report. A blob of prose is the lazy option and it costs the parent
tokens for low information. Better: ask for structure. What was found, where (file +
line), what's uncertain, what wasn't checked. If you want to be strict about it, use
structured outputs (`output_config.format` with a JSON schema) on the subagent's final
turn so the parent gets a typed object.

**Toolset:** default to read-only. A subagent that can write is a subagent that can
conflict with its siblings and surprise its parent. Make write access an explicit,
deliberate choice per spawn.

**Depth:** cap it at one level. A subagent that can spawn subagents produces
exponential cost and untraceable behaviour. Hosted platforms enforce exactly this limit;
so should you. Just don't put `task` in the child's registry.

**Budgets:** the child needs its own turn limit and token ceiling, and the parent should
account for the child's usage in its own totals. Otherwise your cost logging lies.

**Observability:** this is where a naive implementation gets painful. When something goes
wrong you need to know which agent did what. Tag every log line with an agent id and a
parent id. You'll want it within the first hour.

---

## 5.3 Build it

1. Refactor your loop into a function taking `(ctx, messages)` rather than reading
   globals. If you passed `ctx` explicitly in stage 2, this is already done — that's why
   stage 2 said to.
2. Add a `task` tool: two-field schema, spawns a child with a read-only registry and no
   `task` tool of its own.
3. Thread the abort signal so Ctrl-C kills the parent *and* all children.
4. Log with agent ids. Render child activity indented or dimmed so you can follow it.
5. Aggregate token usage from children into the parent's totals.
6. Run several `task` calls from one turn concurrently, and compare wall-clock against
   serial.
7. Optional but worth it: make the child's final answer a structured object via
   `output_config.format`, and have the parent consume the fields.

### Prompts to try

1. `"is our error handling consistent across the API routes?"` — good subagent fit.
   Compare the parent's final token count against doing it inline. Record both.
2. `"read package.json and tell me the version"` — watch whether it spawns a subagent
   for something trivial. If it does, that's your cue for a spawn-discipline instruction.
3. `"compare how modules A and B handle config"` — two independent subagents, ideally
   concurrent.

---

## 5.4 Break it on purpose

1. **Give the subagent a one-line brief** ("look at auth"). Read what comes back. Then
   give it a full brief for the same question and compare. This is the failure mode you
   will actually hit, and the difference is stark.
2. **Let the subagent inherit write tools** and run two concurrently on overlapping
   files. Observe the conflict.
3. **Allow recursion** (put `task` in the child registry) and give it a broad task. Watch
   the cost. Kill it early.
4. **Don't aggregate child usage.** Note that your cost logging now understates by
   multiples, and that you'd have shipped it that way.
5. **Return unstructured prose** from a subagent doing a multi-file audit, then ask the
   parent a follow-up question about a detail. Watch it re-spawn to re-derive something
   that was lost in the summary. That's the double-summarisation cost, felt.

---

## 5.5 Checkpoint

- [ ] `task` spawns an isolated child with its own message array and read-only tools.
- [ ] You can show, with numbers, that the parent's context stayed small compared to
      doing the same investigation inline.
- [ ] Ctrl-C kills parent and children.
- [ ] Child token usage rolls up into parent totals.
- [ ] Recursion is impossible by construction, and you can say why that matters.

**Questions to be able to answer without looking:**

- What's the primary benefit of a subagent, and what's the commonly-cited wrong answer?
- Name three costs of delegation.
- Why must a brief be self-contained, and what happens when it isn't?
- Why should verification usually happen in the main loop rather than a reviewer subagent?
- Why cap delegation depth at one?

Next: [Stage 6 — Sessions & memory](06-sessions-and-memory.md).

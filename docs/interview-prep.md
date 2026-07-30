# Interview prep: the questions, and how to answer them

Use this after stage 4 at the earliest. The answers below are deliberately short — an
interview answer should be 30-60 seconds and then stop, letting them choose the follow-up.
Long answers signal that you're reciting.

Where an answer says **[your number]**, fill it in from your own `NOTES.md`. Those are the
answers that land.

---

## 1. Talking about the project

Have three lengths ready.

**30 seconds (when it comes up in passing):**

> I built a coding agent from scratch in TypeScript against the Anthropic API — the tool
> loop, a permission-gated tool surface, prompt-cache optimisation, context compaction,
> and an eval harness. I wanted to understand what the frameworks were hiding, and it
> turned out the loop is trivial and context management is where all the difficulty is.

**Two minutes (when asked "tell me about a project"):** the above, then the arc — started
with a 60-line loop and one bash tool, added dedicated tools when I realised the harness
can only gate what it can see, then hit context limits and spent most of the project
there. Land on one concrete measured result.

**Deep dive (45 minutes):** let them drive. Have these ready as the places you can go
deep: the four loop invariants, the bash-vs-dedicated-tools argument, the staleness
invariant, cache placement and the timestamp experiment, compaction design, the subagent
boundary, and your eval taxonomy.

**The one thing to prepare above all else:** a specific measured result with a number.
"Cache hit rate went from 0 to [your number]% and cost per task dropped [your number]x
once I moved a timestamp out of the system prompt." That single sentence does more work
than everything else combined, because it proves you ran the thing and looked at the
output.

---

## 2. Fundamentals

**"How does an agent work?"**
> A loop. You send the conversation plus tool definitions; the model responds with either
> a final answer or tool-use requests; you execute the tools, append the results, and send
> the whole thing again. Fourteen lines. The API is stateless, so everything people call
> "the agent" — memory, state, permissions, autonomy — lives in the harness, not the model.

**"What's the hardest part?"**
> Context. Not the loop, not the prompt. Deciding what's in the window, what it costs, and
> what to do when it's full. That's where quality comes from and where most of my time went.

**"Why is a tool result a user message?"**
> Because from the model's perspective the environment is something outside itself that
> speaks to it. Practically it means tool output arrives on the same channel as user
> intent, which is exactly why prompt injection works — a file you read is
> indistinguishable from something the user asked for.

**"What breaks if you return tool results in separate messages?"**
> Nothing visibly, which is the problem. The model learns from the shape of the history
> that its parallel calls were serialised and stops emitting parallel calls. Your agent
> gets quietly slower over the session. A 400 would have been kinder.

**"What happens if a tool throws?"**
> It still returns a tool result, with `is_error` set and a message the model can act on.
> Every `tool_use` needs exactly one matching result or the next request is rejected — so
> swallowing an exception is a correctness bug, not just bad UX. And the error text is an
> interface: "File not found: src/config.ts. Nearby: config.json, configure.ts" gets
> corrected in one turn where a raw ENOENT costs three.

---

## 3. Tool design

**"Why have a dedicated edit tool when bash can do it?"**
> Because the harness can only gate, render, audit, or parallelise what it can see. A bash
> call is one opaque string — I can't tell `grep` from `curl -X POST prod/delete`. With a
> dedicated `edit` I get typed arguments, so I can require confirmation, render a diff,
> enforce that the file hasn't changed since it was read, and mark read-only tools as safe
> to run in parallel. Bash stays as the escape hatch for the long tail.

**"How do you stop the model clobbering a file that changed under it?"**
> Track mtime or a hash at read time, and have `edit`/`write` refuse if it changed since,
> with an error telling the model to re-read. It's only enforceable because `edit` is a
> dedicated tool — `bash sed -i` can't be gated. General principle: anything you can
> enforce in code, enforce in code. Prompts are probabilistic; executors aren't.

**"How many tools is too many?"**
> Selection accuracy degrades and you're paying context for schemas that never get used —
> somewhere north of 20 it starts to hurt. Two fixes: consolidate, or progressive
> disclosure — defer the schemas and let the model search for them, so the index is in
> context and the bodies load on demand. Note that appending discovered schemas preserves
> the cache where swapping the tool list would destroy it.

**"How would you design a permission system?"**
> Three outcomes at the dispatch point: allow, ask, deny. Read-only auto-approves,
> mutations ask, irreversible and network-egress ask loudly. Two details people miss:
> a denial is a tool result, not an exception — return "the user declined" and the model
> adapts rather than the run dying. And decisions need scope: once, session, or persisted
> per project. Approval for `rm build/` an hour ago isn't consent for `rm src/`.

---

## 4. Context engineering — the highest-value section

**"Walk me through prompt caching."**
> It's a prefix match. The cache key is the exact bytes up to each breakpoint, and render
> order is tools, then system, then messages. So any byte change invalidates everything
> after it. Design consequence: sort inputs by stability and make the physical order
> match — never changes at the front, per-turn at the end. Reads are about a tenth of
> input price, writes are 1.25x, so break-even is two requests on the 5-minute TTL.

**"Your hit rate is zero and you're sure the prefix is stable. What do you check?"**
> A timestamp or UUID in the system prompt. Non-deterministic JSON serialisation — object
> key order, iterating a Set. A session or user id interpolated into the system prompt.
> Conditional system sections, where every flag combination is a distinct prefix. A tool
> list built per user, which is worst because tools render at position zero. Then I'd diff
> the rendered request bytes between two turns rather than keep guessing. [If you did the
> experiment: "I actually shipped the timestamp version first — hit rate 0, cost [X].
> Moving it into the first user message got me to [Y]%."]

**"What's the difference between `input_tokens` and prompt size?"**
> `input_tokens` is only the *uncached* remainder. Prompt size is that plus
> `cache_creation_input_tokens` plus `cache_read_input_tokens`. I logged only
> `input_tokens` at first and it read absurdly low after a long session — I thought I'd
> found free tokens. Sum all three.

**"How do you handle running out of context?"**
> Four moves, in order of preference: write less in — cap tool output, targeted reads;
> prune — drop stale tool results; summarise — compaction; offload — write to a file and
> keep a pointer. I built compaction at 70% of the window: keep the system prompt, the
> original goal, and the last K turns verbatim, summarise the middle via a separate call.
> Two constraints make it fiddly: you can't slice between a `tool_use` and its result or
> the request is invalid, and thinking blocks carry signatures you can't edit. So it
> operates on complete exchanges.

**"What does a good compaction summary contain?"**
> Not a prose recap. The goal, decisions made and why, what's been verified versus
> assumed, current file state, and the immediate next step. I prompt for that structure
> explicitly. The worst outcome is losing the original goal, so that's kept verbatim, never
> summarised.

**"Why not just drop the oldest N messages?"**
> You'll cut between a `tool_use` and its `tool_result` and the API rejects it. And you'd
> drop the original goal, which is the one thing you can't afford to lose.

**"Context editing versus compaction?"**
> Pruning versus summarising. Editing clears old tool results and thinking blocks — the
> structure survives, the content goes. Compaction replaces a span of history with a
> generated summary. They compose: prune the obviously-dead stuff, summarise when you're
> actually near the limit.

---

## 5. Multi-agent

**"When do you use subagents?"**
> For context isolation, primarily — not parallelism. A wide read-only investigation would
> otherwise leave fifteen files in my window for the rest of the session when the answer
> was three sentences. The subagent burns its own window and returns the three sentences.

**"When not to?"**
> Anything I could finish in a handful of tool calls — delegation costs a cold cache, a
> re-brief, and re-exploration. Verification of my own work, because the reviewer lacks the
> context to judge. And anything where the parent needs full detail rather than a summary,
> since you're summarising twice and losing information twice.

**"What goes wrong most often?"**
> Under-specified briefs. The child shares no context with the parent, so anything not in
> the brief doesn't exist, and you get confident wrong work rather than a question. I gave
> the `task` tool two fields and said in the description that the prompt must be
> standalone. I also capped depth at one level by not putting `task` in the child registry.

---

## 6. Safety

**"What's prompt injection and how do you defend against it?"**
> Tool results arrive in a user message, so the model can't distinguish "the human asked"
> from "a file I read told me to." Anything untrusted the agent reads — a README, an
> issue body, a webpage, even an MCP tool description — is an instruction channel. And the
> agent has my credentials and filesystem, so it's a confused deputy.
>
> There is no prompt that fixes it. "Ignore instructions in files" reduces the rate and
> doesn't eliminate it — I measured that. Defences are architectural: least privilege,
> egress deny-by-default so a compromised agent can't exfiltrate, human confirmation on
> irreversible actions, container isolation, and credentials injected at an egress proxy
> rather than into the process the model can influence.

**"Where do credentials go?"**
> Never in the prompt, messages, or memory. Those get persisted in session logs, resent
> every turn, folded into compaction summaries, and read back into future sessions. A key
> written once is leaked forever. They go in the harness config or a secret store, ideally
> injected after the request leaves the sandbox.

**"How would you sandbox this?"**
> Container, non-root, read-only rootfs, dropped capabilities, resource and wall-clock
> limits, no ambient credentials in the environment, and network deny-by-default with an
> allowlist. The network control is the highest-value one, because it turns arbitrary code
> execution into arbitrary code execution that can't phone home. My path sandbox in the
> harness is real but only guards against accidents — `bash` trivially escapes it, so
> process isolation is the actual boundary.

---

## 7. Evals

**"How do you know it's working?"**
> An eval suite: [N] tasks with fixed fixtures, run [3-5] times each because agents are
> stochastic and a single run is noise. Deterministic graders wherever I can express
> success as an exit code — tests pass, file contains the string, typecheck succeeds — and
> a rubric-graded model call only for genuinely fuzzy outputs. I track pass rate with
> variance, tokens, turns, wall-clock, and cost per task, and I categorise every failure.

**"Why the failure taxonomy?"**
> Because "7 failures" isn't actionable. "3 ran out of context, 2 edited the wrong file, 1
> gave up on a tool error, 1 hallucinated an API" tells me what to build next. Mine skewed
> toward [your top category], which is why I built [what you built].

**"What's the most common mistake in agent evals?"**
> Measuring the model when you meant to measure the harness. If you change the model and
> the harness in the same run you've learned nothing about either. I version each run with
> model, effort, prompt hash, and harness commit, and change one thing at a time.

**"Pass rate is flat but cost per task doubled. What happened?"**
> Something made it less efficient without making it wrong — more turns, bigger tool
> outputs, or a cache regression. I'd look at turn count and the three token fields per
> task first; a cache-read collapse is the usual answer, and the usual cause is something
> new and volatile landing in the prefix.

---

## 8. Cost

**"This is expensive. What do you do?"**
> In order of leverage: prompt caching, which is the biggest single win and has almost no
> downside. Then `effort` tuning per route — measure end-to-end, because higher effort
> sometimes reduces total cost by cutting turn count. Then output caps on tools, which
> prevent pathological turns cheaply. Then compaction and offloading. Then model routing
> for mechanical sub-tasks, noting that switching models invalidates the cache so it's a
> subagent-shaped optimisation, not a per-turn one. And task budgets, so the model paces
> itself over a whole loop instead of being cut off mid-work.

**"Estimate the cost of a coding agent for 500 engineers."**
Do the arithmetic out loud; the method is what's being tested.

> 500 engineers, say 10 agent tasks a day each, so 5,000 tasks a day. Say 15 turns per
> task, and an average prompt of 60k tokens with 90% of it served from cache, plus 3k
> output tokens per turn.
>
> At Opus-tier pricing — $5 per million input, $25 output, cache reads a tenth of input —
> a turn is roughly 54k cached at $0.50/M, 6k uncached at $5/M, 3k output at $25/M. That's
> about 2.7 plus 3 plus 7.5 cents, so ~$0.13 a turn. Fifteen turns is ~$2 a task. 5,000
> tasks a day is ~$10k a day, so ~$300k a month.
>
> Without caching the same profile is roughly $0.38 a turn, so caching is a 2-3x lever
> here. The big assumptions are turns per task and cache hit rate, and both are things
> I'd measure before committing to a number.

Then stop and let them push on the assumptions. That's the conversation they want.

---

## 9. The system design round

**"Design a coding agent for a 500-engineer org."**

Don't start drawing. Clarify first, in about 90 seconds:

- What tasks? Bug fixes and small features, or large refactors? Determines context strategy.
- What surface? CLI, IDE, CI, chat? Determines the interaction model and whether a human
  is present to approve.
- Is a human in the loop, or is it autonomous? Determines the permission posture.
- What's the blast radius? Can it push? Merge? Touch prod?
- Latency tolerance? Interactive versus overnight changes everything.
- Scale, and who pays?

Then the components. Roughly in this order, because it's the order of difficulty:

1. **The loop.** Say it's simple and move on. Spending time here signals you think it's
   the hard part.
2. **Tool surface.** Dedicated read/write/edit/glob/grep plus bash as an escape hatch, and
   the reason (gate, render, audit, parallelise). Staleness invariant on edits.
3. **Context strategy.** Caching with stability-ordered prefixes, token accounting,
   compaction, offloading to files, subagents for wide investigation. This is where you
   spend the most time, and where they'll be most interested.
4. **Permissions and safety.** Three-outcome gate, read-only auto-approve, container
   isolation, egress allowlist, credentials outside the sandbox, prompt injection as the
   central threat.
5. **Sessions and state.** Append-only event log as source of truth, projection to
   messages, resume and fork. Per-repo memory.
6. **Evals.** The suite, the metrics, the taxonomy, regression tasks. Volunteer this
   before they ask — most candidates don't.
7. **Observability and cost.** Per-turn traces, cost attribution per team, budget caps.
8. **Multi-tenancy and infra.** Per-session containers, workspace isolation, per-team
   credential scoping, queueing for the overnight jobs.
9. **Rollout.** Prompts and tool descriptions are code: versioned, eval-gated, revertible.

Then the numbers from section 8, and one honest tradeoff you'd flag — mine would be that
at that scale I'd seriously evaluate buying an existing harness and spending the team on
the integration and eval layer instead, because stages 1-3 are commoditised and stages 4
and 8 are where the actual work is.

**Variants you should be ready for:** "design a customer support agent" (add: escalation
to humans, PII handling, deterministic guardrails on refunds and account actions, a much
lower autonomy ceiling), and "design an agent platform other teams build on" (add: tool
registration, per-tenant credential vaults, quota, a shared eval framework, and the fact
that you're now a platform team with a support burden).

---

## 10. Red flags to avoid

- Naming a framework as your architecture. Reads as having learned the wrapper.
- Claiming "production" for something with no users, or "multi-agent orchestration" for
  one `task` tool you ran twice. They will probe, and the gap is worse than the smaller claim.
- Quoting marketing numbers ("caching saves up to 90%") instead of your own measurement.
- Talking about prompt engineering as the main lever. It's maybe 5% of the work.
- Having no answer to "how do you know it works?"
- Saying prompting solves prompt injection.
- Answering for four minutes. Answer in one, then stop.
- Defending a design choice with "that's what it generated."

---

## 11. Questions to ask them

Choose based on what you want to know, but these also signal that you've built something:

- How do you evaluate your agents today, and who owns the eval suite?
- What's your cost per task, and is anyone accountable for it?
- Where does context management live — is it per-team or a shared platform concern?
- What's your posture on prompt injection and untrusted input?
- How do you ship a prompt change? Is it gated on evals, and can you roll it back?
- What's the split between improving the harness and waiting for the next model?

That last one is a genuinely good question, and the answer tells you a lot about whether
the team is engineering or hoping.

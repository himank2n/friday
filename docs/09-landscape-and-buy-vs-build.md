# Stage 9 — Landscape: what you'd choose not to build

No milestone. This is 30 minutes of reading, and it's the stage that makes you sound
senior rather than enthusiastic. The judgement questions in an interview are all here.

---

## 9.1 The prior question: should this be an agent at all?

Before the architecture question there's a tier question. Four tiers, increasing in cost,
latency, and variance:

| Tier | Shape | Use when |
|---|---|---|
| **Single call** | One request, one response | Classification, extraction, summarisation, Q&A over provided context |
| **Workflow** | You write the control flow; the model fills steps | The steps are known in advance. Deterministic orchestration, model-powered stages |
| **Agent** | The model decides the control flow | The path can't be specified up front |
| **Multi-agent** | Agents coordinating agents | Genuinely independent tracks at scale |

**Most things labelled "agent" should be workflows.** If you can enumerate the steps, a
workflow is cheaper, faster, more debuggable, and more reliable. The agent tier buys you
one thing — the model choosing what to do next — and you pay for it in tokens, latency,
and non-determinism.

Four criteria before you reach for the agent tier. All four must hold:

1. **Complexity.** Is the task genuinely hard to specify in advance? "Turn this design
   doc into a PR" qualifies. "Extract the title from this PDF" does not.
2. **Value.** Does the outcome justify 10-100x the cost and latency of a single call?
3. **Viability.** Is the model actually good at this class of task? Test before you build.
4. **Cost of error.** Can mistakes be caught and reverted? Tests, code review, a git
   revert, a human approval gate. An agent that irreversibly touches production money
   with no gate is a bad design regardless of how good the model is.

Being able to say "we looked at this and it should be a workflow, here's why" is worth
more in an interview than any amount of agent enthusiasm.

---

## 9.2 The four ways to build one

The useful axes are **who owns the harness** (the loop plus context management) and **who
owns the deployment** (the infrastructure it runs on). These get conflated constantly
because three of the four options own the harness but not the deployment.

| Approach | You write | Harness | Deployment | Tools available |
|---|---|---|---|---|
| **Manual loop** | The loop itself | You | You | Only what you define |
| **Tool Runner** (SDK helper) | Just the tool functions | SDK | You | Only what you define |
| **Agent SDK** (Claude Code as a library) | A prompt plus options | SDK, with built-in tools | You | Read/Write/Edit/Bash/Glob/Grep/Web + MCP + subagents |
| **Managed Agents** | Agent config plus your custom tool results | Provider | **Provider** — hosted per-session sandbox | Hosted toolset + skills/MCP + your tools |

Notes that matter:

- **Tool Runner is not the Agent SDK.** Different packages, constantly confused. The Tool
  Runner is a thin helper inside the regular SDK (`client.beta.messages.tool_runner`) that
  drives the request → execute → loop cycle over *tools you define*. No built-in tools, no
  filesystem, no sandbox. The Agent SDK is Claude Code packaged as a library: built-in
  tools, context management, hooks, subagents, permissions, sessions.
- **"I need fine-grained control" is rarely a reason to hand-write the loop.** The Tool
  Runner exposes per-turn hooks — you can gate a call before it executes, intercept
  errors, modify results, retry a turn. Approval gates and logging do not require a
  manual loop. Write the manual loop when you want to own the loop (which, for learning,
  is exactly the right reason — that's this curriculum), or when your control flow doesn't
  fit the hooks.
- **Managed Agents is the only option that also hosts the sandbox.** That's the real
  distinction. If the alternative is you building container orchestration, network
  policies, and session persistence, "hosted" is often the *simpler* option even though
  it's a bigger platform. Its natural fits: scheduled runs, long-lived sessions,
  versioned agent configs, and anything where you'd otherwise write a scheduler.

**What you'd actually pick, in practice:**

- Learning, or a genuinely custom control flow → manual loop.
- A production custom-tool agent on your own infra → Tool Runner.
- A coding/filesystem agent, quickly, on your own infra → Agent SDK. Don't rebuild
  Claude Code.
- Hosted, scheduled, stateful, multi-tenant, or "I don't want to own the sandbox" →
  Managed Agents.

Having built the manual loop is what lets you evaluate the other three, which is the
whole argument for this curriculum.

---

## 9.3 On frameworks

You were told at the start not to install LangChain, CrewAI, AutoGen, or LlamaIndex.
Here's the honest position now that you've built the thing yourself.

**What they hide:** the loop (which is 14 lines), the tool schema plumbing (real but
small), and provider differences (real if you're multi-provider). What they *also* hide
is context management, cache placement, and token accounting — which are the parts that
actually determine whether your agent is good. When something goes wrong there, you're
debugging through an abstraction that was designed for a demo.

**When they're reasonable:** genuine multi-provider requirements, an org standard you
don't get to relitigate, prototyping speed where you'll throw the code away, or a
retrieval pipeline where the framework's document loaders save real time.

**The interview angle:** naming a framework as your answer to "how would you build this"
reads as having learned the wrapper. Saying "the loop is small enough that I'd own it, and
here's what I'd use a library for" reads as having built one. That distinction is why this
curriculum exists.

---

## 9.4 The buy-vs-build conversation you'll actually have at work

Someone will propose building an internal agent. The useful questions, roughly in order:

1. **Should it be a workflow?** (9.1. Usually yes.)
2. **What's the eval?** If nobody can say how we'd know it works, that's the first
   deliverable, not the agent.
3. **What's the cost per task, and who pays?** Estimate it before building. Agents are
   the first LLM product where cost surprises people.
4. **What's the blast radius?** What can it touch, and what's irreversible?
5. **Is there an existing harness we should use?** Claude Code, Cursor, the Agent SDK, a
   hosted platform. Rebuilding a coding agent to save a licence fee is almost always a
   bad trade — you'd be rebuilding stages 1-8, and stages 4 and 8 are the expensive ones.
6. **What's the maintenance story?** Models change. Prompts are code. Evals are
   infrastructure. Who owns it in a year?

The senior move in that conversation is usually narrowing the scope: one high-value task,
one measurable outcome, one reversible blast radius. Then expand.

---

## 9.5 Where the field is heading, briefly

Worth having a view on, because "where do you think this goes?" gets asked:

- **Context is the bottleneck, not capability.** Windows keep growing, but attention over
  a million tokens of stale tool output is still bad attention. Pruning, summarising, and
  offloading are becoming first-class platform features rather than things every team
  reinvents — the movement of compaction and context editing into the API is exactly that.
- **Harnesses are consolidating.** The loop is commoditised. Differentiation is moving to
  tool surface, evals, and integration depth.
- **Long-horizon autonomy is the frontier.** Single requests running for many minutes;
  agents working overnight. That pushes the hard problems toward verification, budget
  pacing, and progress reporting — which is why stage 8 matters more each year.
- **The security story is immature.** Prompt injection has no clean answer, and the
  industry is converging on architectural mitigation (least privilege, egress control,
  credentials outside the sandbox) rather than model-level fixes. If you want a niche
  with room in it, that's the one.

---

## 9.6 Checkpoint

Nothing to build. You're done when you can answer these cold:

- When should something be a workflow rather than an agent? Give the four criteria.
- What's the difference between the Tool Runner and the Agent SDK?
- Which approach owns the deployment, and why does that make it simpler rather than
  heavier for some use cases?
- Why wouldn't you use LangChain? Give a reason that isn't snobbery.
- Your director wants to build an internal coding agent from scratch. Talk them through it.
- What's the hardest unsolved problem in agents right now?

Next: [interview-prep.md](interview-prep.md).

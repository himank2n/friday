# Build Your Own Coding Agent

A staged curriculum for understanding AI agents from first principles, by building one.

You write all the code. This repo contains the concepts, the invariants, the failure
modes to trigger deliberately, and the checkpoints that prove you understood each layer.
There are no solution files. If you get stuck, the fix is usually in the "Failure modes"
section of the stage you're on.

Working title for the thing you'll build: **friday**.

---

## The thesis

An agent is four things:

```
  a model  +  tools  +  a loop  +  context management
```

That's it. Claude Code, openclaw, Cursor, Devin, and every "agent framework" on GitHub
are engineering around those four. The model is a pure function:

```
  (system prompt, tool definitions, message history) -> next content blocks
```

It has no memory, no state, no ability to execute anything, and no knowledge of what
happened five seconds ago beyond what you put in the request. Every property people
attribute to "the agent" — persistence, autonomy, judgement about permissions, the
ability to run a 40-minute task — lives in **your harness**, not in the model.

Internalising that sentence is 60% of what this curriculum is for. The other 40% is
context engineering (stage 4), which is where agents actually get good or stay bad.

---

## Stages

**Read [docs/00-how-to-study-this.md](docs/00-how-to-study-this.md) first.** It covers what
an agent interview actually tests, why building beats reading, and — the question most
worth getting right — how much of this to write by hand versus delegate to AI. Fifteen
minutes, and it determines how you spend the next three weeks.

Then each stage below is a runnable milestone. Do not skip ahead — stage 4 makes no sense
until you've felt stage 1 break.

| # | Stage | What you build | Core idea |
|---|-------|----------------|-----------|
| 1 | [Foundations](docs/01-foundations.md) | ~60-line agent with one `bash` tool | The loop is the whole idea |
| 2 | [Tools & permissions](docs/02-tools-and-permissions.md) | read/write/edit/glob/grep + a permission gate | Tool surface design is product design |
| 3 | [Streaming & control](docs/03-streaming-and-control.md) | Token streaming, interrupt, cancellation | Agents are long-running processes |
| 4 | [Context engineering](docs/04-context-engineering.md) | Token accounting, prompt caching, compaction | The hard part. The interview differentiator |
| 5 | [Subagents](docs/05-subagents-and-orchestration.md) | A `task` tool that spawns a nested agent | Context isolation, not parallelism |
| 6 | [Sessions & memory](docs/06-sessions-and-memory.md) | Append-only event log, resume, cross-session memory | State belongs on disk, not in RAM |
| 7 | [MCP & extensibility](docs/07-mcp-and-extensibility.md) | An MCP client + a tiny MCP server | How tools stop being per-host |
| 8 | [Evals, safety, production](docs/08-evals-safety-production.md) | An eval harness + a sandbox story | "It worked once" is not evidence |
| 9 | [Landscape](docs/09-landscape-and-buy-vs-build.md) | Nothing — this one is reading | Knowing what you'd *not* build |

Then: [interview-prep.md](docs/interview-prep.md) — the questions you'll actually be
asked about all of this, and how to answer them crisply.

---

## Prerequisites

- Node 20+ (you have 24) or Python 3.11+. **Recommendation: TypeScript.** It's your
  strongest language, and every reference implementation in this space (Claude Code,
  openclaw, the Agent SDK) is Node/TS, so the code you read for comparison will match.
  The concepts are language-independent; where the SDK surface differs I've noted it.
- An Anthropic API key. Two options:
  - `export ANTHROPIC_API_KEY=sk-ant-...` from https://console.anthropic.com
  - or install the CLI (`brew install anthropics/tap/ant`) and run `ant auth login`,
    which stores a profile the SDKs pick up automatically with a zero-arg client.
- **Budget expectation:** stages 1-3 cost single-digit dollars. Stage 4 and stage 8
  (evals) are where you'll spend, because you'll run the same task dozens of times.
  Default to `claude-opus-5`; drop to a cheaper model only for high-volume eval loops
  where you're measuring your harness, not the model.

Do **not** install LangChain, LlamaIndex, CrewAI, AutoGen, or any agent framework.
The entire point is that you build the abstractions yourself so you can see what the
frameworks are hiding. You may use exactly two dependencies: the official
`@anthropic-ai/sdk` and `zod` (for tool schemas, from stage 2).

---

## How to work through this

For each stage:

1. **Read the stage doc.** Read all of it before writing code, including the failure
   modes. The failure modes are the load-bearing part; they're the things you'd
   otherwise spend a day rediscovering.
2. **Build the milestone.** No solution code exists. If your design differs from what
   the doc implies, that's fine as long as the invariants hold.
3. **Break it on purpose.** Each stage lists specific things to deliberately do wrong
   so you see the error the API or the model produces. You will remember these.
4. **Hit the checkpoint.** Each stage ends with a concrete, verifiable check. Not
   "it seems to work" — an actual observation with output you can point at.
5. **Write your own notes.** Keep a `NOTES.md` per stage: what surprised you, what
   the token numbers were, what you'd do differently. That file is your interview prep.

Suggested pace if you're doing this alongside DS&A prep: one stage per session,
stages 1-4 in the first week, 5-9 in the second. Stage 4 deserves two sessions.

---

## Layout you'll end up with

Nothing here is prescriptive — it's the shape most people converge on, so you have a
target to aim at rather than a blank page:

```
friday/
  src/
    minimal.ts          stage 1: the whole agent in one file. Keep it forever, unchanged.
    loop.ts             stage 3+: the real loop, streaming, cancellation
    tools/
      registry.ts       stage 2: definition + dispatch
      bash.ts read.ts write.ts edit.ts glob.ts grep.ts
      permissions.ts    stage 2: allow / ask / deny gate
    context/
      tokens.ts         stage 4: accounting
      cache.ts          stage 4: breakpoint placement
      compact.ts        stage 4: summarisation
    agents/
      subagent.ts       stage 5
    session/
      log.ts            stage 6: append-only JSONL
    mcp/
      client.ts         stage 7
  evals/                stage 8
  NOTES.md
```

Keep `src/minimal.ts` from stage 1 and never refactor it. When someone asks you what an
agent is, that file is your answer, and you'll want to be able to open it.

---

## A note on scope

Stages 1-4 give you a real, usable agent and cover ~everything that shows up in an
interview. Stages 5-8 are what separates a demo from a product. Stage 9 is the "should
we even build this" conversation, which is the one you'll actually have at work.

If you only have time for four stages, do 1, 2, 4, and 8.

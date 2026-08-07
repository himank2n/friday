# How to study this: interview reality, and how much AI to use

Read this before stage 1. It decides how you spend the next three weeks.

---

## 1. What an agent-focused interview actually tests

You will not be asked to write an agent loop from scratch. Nobody whiteboards
`while (stop_reason === "tool_use")`. Here's the realistic shape of a loop for an
agent/AI-infrastructure role (Anthropic, OpenAI, Cursor, Sierra, Harvey, or an AI team
inside a big company):

| Round | What it is | Does building this help? |
|---|---|---|
| Coding | Standard DS&A, or a practical coding exercise | No. Prep separately |
| System design | Often agent-flavoured: "design a coding agent", "design an agent platform for 500 engineers", "design an LLM-powered support agent" | **Enormously** |
| Deep dive / experience | They probe whatever you claim. If your resume says "built an agent," expect 45 minutes on it | **This is the whole point** |
| Practical / take-home | Build something against an LLM API. Frequently AI-assisted-coding is expected, not banned | Yes, directly |
| Product / judgement | "When would you *not* use an agent?" "How do you know it's working?" | Yes — stages 8 and 9 |
| Behavioural | Standard | Indirectly: gives you a real project to talk about |

The knowledge is tested. The typing is not.

### What they're screening for, in priority order

1. **Do you understand that the model is stateless and the harness is the product?**
   This is the dividing line. People who've only used agents talk about the model.
   People who've built one talk about context, tools, and the loop.
2. **Can you reason about context as a budget?** The single most differentiating topic.
   Most candidates have never measured a cache hit rate or written a compaction step.
3. **Do you know the failure modes?** Not "LLMs hallucinate" — specific things:
   orphaned tool results, silent cache invalidation, parallel-call degradation from
   split result messages, stale-file clobbering, prompt injection through tool output.
4. **Can you talk about cost and latency numerically?** Agents are expensive. Anyone
   shipping one has fought this. If you haven't, it shows immediately.
5. **Do you know when not to build one?** Stage 9. Judgement reads as seniority.
6. **Do you have evals?** "It worked when I tried it" is the answer of someone who
   hasn't shipped.

### What they are not screening for

- Knowing framework APIs. LangChain fluency is close to worthless, and in some rooms
  actively negative, because it signals you learned the wrapper and not the mechanism.
- Memorised parameter names. Nobody cares whether you remember that `effort` lives
  inside `output_config`. They care that you know what it trades off.
- Novel research. You're being hired to build systems around models, not to train them.

---

## 2. Why building beats reading

You could read all nine stage docs in an afternoon and be able to *describe* everything
in them. That gets you through the first layer of questions and falls apart on the
second, because the second layer is always "tell me about a time it broke" or "what did
you measure?"

Concretely, the things you can only get by building:

- **Numbers.** Your cache hit rate before and after. Tokens per task. Cost per run.
  Turn counts. What compaction saved. These are the details that make an answer
  believable, and you can't invent them convincingly.
- **War stories.** The 400 you got from an orphaned `tool_result`. The afternoon lost to
  a timestamp in a system prompt. The silent degradation from splitting parallel results.
  Interviewers are trained to dig for these, and the absence of them is loud.
- **Design opinions you can defend.** "Why does `edit` require a unique match?" has a
  crisp answer if you shipped the strict version after watching the loose version
  silently corrupt a file. It has a vague answer if you read about it.
- **Calibration.** You'll learn that context management is 60% of the work and prompt
  wording is 5%, which is the opposite of what the internet implies. That calibration
  shows up in every answer you give.

Rough split: reading gets you ~60% of the interview surface. Building gets you the rest,
and upgrades the first 60% from recited to owned.

---

## 3. How much AI to use

This is the real question, and "none" is the wrong answer — you'd spend 50 hours and
learn nothing extra from writing a glob wrapper by hand.

### The rule

> **Hand-write the code whose behaviour you need to be able to defend under
> questioning. Delegate everything else.**

The corollary, and this is the one that matters:

> **Do not commit a line into this project you can't give the reason for.**

If a deep-dive asks "why did you structure it that way?" and the honest answer is "that's
what it generated," the round is over. Not because using AI is bad — because you claimed
a project you don't own.

### Per stage

| Stage | Hand-write | Delegate to AI |
|---|---|---|
| 1 Foundations | **All of it.** 60 lines. This file is your answer to "how does an agent work" | Nothing |
| 2 Tools | Registry design, permission gate, staleness check, path sandbox | glob/grep/read wrappers, output truncation, line numbering, error message polish |
| 3 Streaming | The interrupt state machine (all three cases). It's a live interview question | Event-to-renderer plumbing, ANSI colours, spinner |
| 4 Context | **All of it.** Especially: run the caching experiments yourself and read the raw `usage` numbers with your own eyes | Nothing. This is the differentiator |
| 5 Subagents | The orchestration boundary: what goes down, what comes back, why | Nothing much — it's small |
| 6 Sessions | Replay/resume validity logic | JSONL read/write, file layout, CLI flags |
| 7 MCP | Registry merge, name-collision handling. Read the spec yourself | JSON-RPC transport, stdio framing, the sample server |
| 8 Evals | The harness and the choice of metrics — deciding what to measure *is* the skill | Task fixtures, the repos/files your eval tasks operate on, report formatting |
| 9 Landscape | Reading only | — |

Rough time: fully by hand, 40-60 hours. With that split, 20-25 hours and the same
learning, because the hours you cut were the ones teaching you nothing.

### Three techniques that make AI a study aid instead of a shortcut

**Use it as a Socratic partner, not a producer.** The highest-value prompts here are not
"write me a tool registry." They're:

- "I'm designing the permission gate this way. What breaks at scale? What am I not
  considering?"
- "Explain why the API rejects an assistant message with a `tool_use` that has no
  matching result. What's the underlying constraint?"
- "Critique this compaction strategy. What information will it lose that matters?"
- "Argue the opposite: why *shouldn't* `edit` require a unique match?"

**Close the tab and re-implement.** After AI writes something you decided to delegate,
if it's design-bearing, delete it and write it from memory. If you can't, you don't own
it, and now you know. This is the cheapest ownership test there is.

**Write the notes yourself.** `NOTES.md` per stage — what surprised you, the numbers,
what you'd change. Do not generate this. It is literally your interview prep, and the
act of writing it is where the consolidation happens.

### The anti-pattern

Vibe-coding the whole thing in an afternoon, ending up with a working agent, and
believing you've learned agents. You'll have learned that Claude can write an agent.
The tell is that you'll be unable to answer "what did you try that didn't work?" —
because nothing didn't work, because you never made a decision.

---

## 4. Which language

Either TypeScript or Python works. Everything in stages 1-9 is language-independent — the
loop, the invariants, cache placement, compaction, evals. Every SDK is generated from the
same OpenAPI spec, so it is the same JSON on the wire either way. Concepts transfer at
100%, syntax in a day. **Don't spend more than ten minutes on this decision.**

First, the honest version of "isn't everything Python?" — it depends which population you
count:

| Python dominates | TypeScript dominates |
|---|---|
| ML/AI research, anything model-adjacent | Coding agents and dev tooling |
| RAG and data pipelines | Anything shipping in an editor, CLI, or browser |
| Frameworks and tutorials (LangChain, LlamaIndex, CrewAI, AutoGen, DSPy, Pydantic AI) | Claude Code, Claude Agent SDK, Cursor, Cline, Continue, Vercel AI SDK, Mastra |
| Enterprise AI-platform teams grown out of data science | Product companies whose app layer is already JS |

Count every agent in existence and Python has a plurality, largely on the strength of
tutorials and the ML ecosystem. Count *coding* agents and it flips to TypeScript. "Most
agents are Python" is technically true and misleading for this particular project.

**Decide from your target roles, not from the general trend.** Read five real job
descriptions for roles you'd actually take and count which language appears in the
requirements. That's better evidence than any argument here.

- **Choose TypeScript if** it's your strongest language and you're aiming at dev tools or
  AI product engineering. Two concrete advantages: zero language friction means your whole
  cognitive budget goes to the concepts (which matters most in stage 4), and the reference
  implementations you'll read for comparison at stage 9 are TS.
- **Choose Python if** you're aiming at ML-adjacent teams — research-adjacent, ML infra,
  data platform — where the language is part of the signal, or if you want the eval and
  observability tooling ecosystem, which skews Python-first.

### If you go Python

Substitutions from what the stage docs assume:

| Stage doc says | Python equivalent |
|---|---|
| `@anthropic-ai/sdk` | `anthropic` |
| `zod` | `pydantic` |
| `client.messages.stream()` + `finalMessage()` | `client.messages.stream()` + `get_final_message()` |
| Structured output via `zodOutputFormat` | `client.messages.parse(output_format=Model)` |
| Tool Runner (`betaZodTool` + `toolRunner`) | `@beta_tool` decorator + `client.beta.messages.tool_runner` |

Two genuine differences, not just naming:

- **The Python SDK refuses non-streaming requests with large `max_tokens`** and raises
  rather than letting you hit a timeout. So stage 3's streaming milestone effectively
  arrives during stage 1. That's fine — arguably better — just know why it happened.
- **Pin to Python 3.12 or 3.13, not 3.14.** 3.14 is new enough that SDK and tooling
  support lags, and debugging a dependency resolution problem is not what you're here to
  learn. Check that `anthropic` installs cleanly before you commit to a version.

---

## 5. What to put on a resume, and what to claim

Once you've done stages 1-4 and 8, the honest and strong framing is something like:

> Built a coding agent from scratch (TypeScript or Python, Anthropic API): tool-use loop with
> parallel execution, permission gating, filesystem sandboxing, prompt-cache
> optimisation (measured hit rate X%, cost reduction Y%), context compaction to run
> past the window limit, and an eval harness over N tasks.

Every clause there is defensible for 10 minutes if you built it. Do not add clauses you
can't defend for 10 minutes. Specifically: don't claim multi-agent orchestration if you
built one `task` tool and ran it twice, and don't claim "production" for something with
no users.

Note what's load-bearing in that sentence: **the measured numbers and the eval harness.**
Those are the two things most candidates don't have.

---

## 6. Sequencing against the rest of your prep

Given a Sept 7 leave-end date and DS&A plus system design also competing for time:

- This is **not** the highest-priority item. Coding rounds gate everything, and they're
  the most trainable. Protect that time first.
- Stages 1-4 are the high-leverage core: roughly 12-15 hours with the AI split above,
  and they cover most of what an interview touches.
- Stage 8 (evals) is disproportionately valuable per hour, because almost nobody has it.
  Do it even if you skip 5-7.
- Stage 9 is 30 minutes of reading and gives you the judgement answers.
- If you're time-boxed to one weekend: stage 1, stage 2 (skeleton only), stage 4 in full,
  stage 8 in miniature. That's a defensible project.

Next: [Stage 1 — Foundations](01-foundations.md).

# Stage 1 — Foundations: the model, the primitives, and the loop

**Milestone:** a single file, roughly 60 lines of real logic, that can answer
"how many TypeScript files are in this repo, and which is the largest?" by running
shell commands. No framework. No abstractions. One tool.

**Time:** one session. Most of it reading, not typing.

---

## 1.1 The one API you need

Everything goes through a single endpoint: `POST /v1/messages`. Tool use, streaming,
structured output, caching, thinking — all of it is parameters on that one call. There is
no separate "agents API" you're missing.

The request is, minimally:

```
{
  model:      "claude-opus-5",
  max_tokens: 16000,
  system:     "...",           // optional, string or array of text blocks
  tools:      [ ... ],         // optional
  messages:   [ ... ]          // the conversation so far
}
```

The response is a `Message` object. The parts that matter to you:

| Field | What it is |
|---|---|
| `content` | An **array of content blocks**, not a string |
| `stop_reason` | Why generation ended. This drives your control flow |
| `usage` | Token counts. You will care about these a lot in stage 4 |
| `model` | Which model actually served it |

### The API is stateless

This is the single most important structural fact. There is no session on the server.
Every request carries the entire conversation. Turn 40 of a conversation resends turns
1-39 verbatim. The consequences ripple through everything:

- Cost and latency grow with conversation length, roughly quadratically over a session
  (each turn resends a longer history). Prompt caching (stage 4) exists to blunt this.
- "The agent remembers X" means "you put X in the messages array."
- Crash recovery is just replaying a message array (stage 6).
- Anything you want the model to forget, you delete from the array. There is no other
  mechanism.

### Content blocks

`content` is a heterogeneous array. The types you'll meet in this stage:

```
{ type: "text",     text: "..." }
{ type: "thinking", thinking: "...", signature: "..." }   // reasoning
{ type: "tool_use", id: "toolu_...", name: "bash", input: { command: "ls" } }
```

And the one *you* produce, in a user message:

```
{ type: "tool_result", tool_use_id: "toolu_...", content: "...", is_error?: true }
```

Note the shape of the conversation this implies. A tool call is **two messages**: an
`assistant` message containing the `tool_use` block, then a `user` message containing
the `tool_result`. The tool result is attributed to the *user* role, because from the
model's perspective the environment is something outside itself that speaks to it. That
framing matters in stage 8 when we talk about prompt injection: tool output arrives on
the same channel as user input, which means it carries user-level trust, which means
a file you read can contain instructions.

### `stop_reason` is your control flow

| Value | Meaning | What you do |
|---|---|---|
| `end_turn` | Model finished naturally | Exit the loop, show the text |
| `tool_use` | Model wants tools run | Execute them, feed results back, loop |
| `max_tokens` | Hit your output cap | Raise `max_tokens` or stream; the response is truncated mid-thought |
| `stop_sequence` | Hit a custom stop string | Rare in agents |
| `pause_turn` | A server-side tool hit its internal iteration limit | Resend the conversation as-is to resume. Do **not** append "continue" |
| `refusal` | Safety classifiers declined | `content` may be empty or partial. Check this **before** reading `content[0]` |
| `model_context_window_exceeded` | You blew the context window, not the output cap | Compact (stage 4) |

Write your loop as a switch on `stop_reason` from the very first version. The most
common beginner bug in this space is `if (response.content[0].type === 'tool_use')`,
which breaks the moment a thinking block or a text block comes first — which is most
of the time.

### Model parameters worth knowing now

- `model: "claude-opus-5"` — the current default. Exact string, no date suffix.
- `max_tokens` — a **hard cap on thinking plus response text combined**. On Opus 5
  thinking is on by default, so a `max_tokens` you sized for text alone will truncate.
  Use 16000 for non-streaming, 64000+ once you're streaming (stage 3).
- `output_config: { effort: "low" | "medium" | "high" | "xhigh" | "max" }` — controls
  how much the model thinks and how much work it does per turn. Default `high`.
  `xhigh` is the sweet spot for coding/agentic work; `low` and `medium` are
  surprisingly strong and are your main cost lever.
- `thinking: { type: "adaptive", display: "summarized" }` — the model decides how much
  to think. `display` defaults to `"omitted"`, which means thinking blocks arrive with
  empty text. If you want to *show* reasoning in your UI, you must opt in explicitly.
  There is no fixed "thinking token budget" any more; `budget_tokens` is removed and
  returns a 400.
- `temperature` / `top_p` / `top_k` are **removed** on Opus 5 and return a 400. Steering
  is done with prompting, not sampling knobs. If you've built LLM apps before, this is
  the habit most likely to trip you.
- Assistant-turn prefilling (ending `messages` with an `assistant` message to force a
  format) also returns a 400. Use structured outputs (`output_config.format`) instead.

> **Read the reference, not your memory.** This space moves fast enough that your
> intuitions from six months ago are wrong. When you need a parameter shape, check the
> current API docs rather than pattern-matching from an old tutorial.

---

## 1.2 What a tool actually is

A tool definition is three fields:

```
{
  name:         "bash",
  description:  "Run a shell command and return combined stdout and stderr.",
  input_schema: {                       // JSON Schema, object at the root
    type: "object",
    properties: { command: { type: "string", description: "The command to run" } },
    required: ["command"]
  }
}
```

That's the whole contract. Now the part that people take too long to internalise:

**The model cannot execute anything.** It emits a `tool_use` block, which is a
structured request. Your code decides whether to honour it, how, with what privileges,
and what to report back. You could return a lie; the model would believe it. The tool
name doesn't need to correspond to anything real. Nothing is sandboxed by default
because nothing is *executed* by anyone but you.

This is empowering and dangerous in equal measure, and it's the source of most of the
interesting engineering:

- Permission gates work because you're the one running the command (stage 2).
- The read-before-write invariant works because you can refuse a write (stage 2).
- Prompt injection is dangerous because your executor is doing what the model asked,
  and the model may be doing what a malicious file asked (stage 8).

**The description is the interface.** The model chooses tools based almost entirely on
`description` plus the property descriptions. A tool that "doesn't get called" is
usually a description problem, not a model problem. Be prescriptive about *when* to
call it, not just what it does: "Call this when the user asks about the contents of a
file" beats "Reads a file."

---

## 1.3 The loop

Here is the entire idea of agents, in pseudocode. Read it, then close this file and
write it yourself.

```
messages = [ { role: "user", content: userInput } ]

loop forever:
    response = api.messages.create({ model, max_tokens, tools, system, messages })

    append { role: "assistant", content: response.content } to messages   # VERBATIM

    if response.stop_reason == "refusal":  handle and exit
    if response.stop_reason == "pause_turn": continue          # resend as-is
    if response.stop_reason != "tool_use":  print text; exit

    results = []
    for each block in response.content where block.type == "tool_use":
        output = execute(block.name, block.input)              # your code, your rules
        results.append({ type: "tool_result", tool_use_id: block.id, content: output })

    append { role: "user", content: results } to messages      # ALL results, ONE message
```

Fourteen lines. Everything else in this curriculum is refinement of those fourteen
lines. When an interviewer asks "how does an agent work," this is the answer, and the
follow-up questions are all about the words in capitals.

### The four invariants

Violate any of these and you get either a 400 or silently degraded behaviour.

1. **Append `response.content` verbatim.** Not `response.content[0].text`. Not a
   reconstructed object. The array as returned, including `thinking` blocks with their
   `signature` field intact, including blocks whose text is empty. The API validates
   thinking-block signatures; editing or dropping them breaks the turn. This is also
   why stage 4's compaction needs care — you can't naively rewrite history.

2. **Every `tool_use` gets exactly one `tool_result`, with a matching `tool_use_id`.**
   If the model asked for three tools and you return two results, the request is
   rejected. There is no partial-response mode.

3. **All results for one assistant turn go in a single user message.** If the model
   emits three parallel `tool_use` blocks and you send back three separate user
   messages, you don't get an error — you get something worse. The model learns from
   the shape of the history that its parallel calls were serialised, and it stops
   emitting parallel calls. Your agent quietly gets slower over the session.

4. **Failures are results, not exceptions.** A tool that throws still returns a
   `tool_result`, with `is_error: true` and a useful message. Swallowing the error and
   returning nothing violates invariant 2. Returning a stack trace with no explanation
   wastes a turn. Return something the model can act on: "File not found: src/foo.ts.
   The directory src/ contains: bar.ts, baz.ts."

### Loop termination

Your loop as written above runs forever if the model keeps calling tools. Add a bound
now, not later: a max-turn count (start with 20) and a wall-clock deadline. Real agents
also add a token budget. When you hit the bound, say so in the output rather than
silently stopping — a truncated agent run that *looks* complete is the worst failure
mode there is.

---

## 1.4 Build it

**Milestone: `src/minimal.ts`.** One file. One tool (`bash`). A `main()` that takes a
prompt from `process.argv` and prints the final text.

Constraints, chosen to force you to feel the primitives:

- No streaming yet. Non-streaming responses only.
- No `zod`. Hand-write the JSON Schema. You should feel the friction so that stage 2's
  schema generation feels like a solution to a problem you actually had.
- Print, to stderr, on every iteration: turn number, `stop_reason`, the tool names
  requested, and `usage.input_tokens` / `usage.output_tokens`. You are building an
  observability habit now, and you'll need those numbers in stage 4.
- Handle all seven `stop_reason` values explicitly, even the ones you think can't
  happen. When one fires unexpectedly you'll want the log line.

For `bash`, run the command and return combined stdout and stderr, with a timeout
(10s is fine) and a truncation cap on output (start with 30,000 characters and note
what happens when you hit it). Do not build a sandbox yet; that's stage 8. Do not run
this against anything you care about.

### Prompts to try, in order

1. `"how many .ts files are in this directory tree?"` — one or two tool calls. Baseline.
2. `"what's the largest file in this repo, and how many lines is it?"` — multi-step;
   watch the model chain commands.
3. `"summarise what this project does"` — watch it explore. Note how many turns and how
   many tokens. Write the numbers in NOTES.md; you'll compare after stage 4.
4. `"count the files, and also tell me the current git branch"` — two independent
   questions. Does it emit two `tool_use` blocks in one response? That's parallel tool
   use, and it's on by default.

---

## 1.5 Break it on purpose

Do all five. Each one takes two minutes and teaches you something you'd otherwise
debug at 11pm in six months.

1. **Drop a tool_result.** With a prompt that triggers parallel calls, return only the
   first result. Read the 400 carefully. This is invariant 2.

2. **Split parallel results across two user messages.** No error. Now run several
   multi-part prompts and watch whether the model keeps parallelising. This is
   invariant 3, and the fact that it fails *silently* is the lesson.

3. **Append only the text, not the full content array.** Strip everything except text
   blocks from the assistant message before appending. You'll get either a 400 about
   missing `tool_use` blocks or a model that's confused about what it just did.
   This is invariant 1.

4. **Set `max_tokens: 200`.** Observe `stop_reason: "max_tokens"` and note that
   `content` is truncated mid-sentence. Now consider: your loop appended that truncated
   assistant turn to history. What does the next turn look like? This is why
   `max_tokens` is a correctness concern, not just a cost knob.

5. **Return a lie.** Hard-code your `bash` executor to always return
   `"command not found"`. Watch the model try three alternative commands, then give up
   and tell the user the tool is broken. Then hard-code it to return a plausible but
   false answer ("47 files") and watch the model confidently report it. There is no
   verification layer you didn't build. This is the most important two minutes in
   stage 1.

---

## 1.6 Checkpoint

You're done with stage 1 when all of these are true and you can show output for each:

- [ ] `node src/minimal.ts "what's the largest file in this repo"` produces a correct
      answer via at least two tool calls, and your stderr log shows the turn-by-turn
      `stop_reason` and token counts.
- [ ] You have observed a single assistant response containing two or more `tool_use`
      blocks, and your loop handled them in one user message.
- [ ] You can state, from memory, why `tool_result` uses the `user` role.
- [ ] You have triggered a 400 by violating invariant 2, and you know what the error says.
- [ ] Your NOTES.md records token counts for prompt 3 above.
- [ ] Your loop has a turn bound and reports when it hits it.

**Questions to be able to answer without looking:**

- Where does agent "memory" live?
- What are the two messages that make up a single tool call, and what role does each have?
- What happens if you return tool results in separate messages, and why is that worse
  than an error?
- Why is `max_tokens` a correctness issue on Opus 5 specifically?
- If the model asks to run `rm -rf /`, what stops it?

Next: [Stage 2 — Tools & permissions](02-tools-and-permissions.md).

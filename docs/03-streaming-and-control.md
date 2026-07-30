# Stage 3 — Streaming & control: agents are long-running processes

**Milestone:** the loop streams tokens as they arrive, you can interrupt a run with
Ctrl-C and leave the conversation in a valid state to continue from, and a single turn
can safely request 64,000 output tokens.

**Time:** half a session. Conceptually small, but the interrupt semantics are subtle and
they're the part interviewers poke at.

---

## 3.1 Why streaming stops being optional

Three separate reasons, and only one of them is UX:

1. **UX.** A 90-second silent pause reads as a hang. This is the obvious one.
2. **HTTP timeouts.** Non-streaming requests with large `max_tokens` hit the SDK's
   request timeout (default 10 minutes) and connection-idle limits before the model
   finishes. Above roughly 16,000 `max_tokens`, non-streaming is a latent failure. The
   Python SDK will actually refuse and raise rather than let you set up the timeout.
   Streaming keeps bytes flowing, so the connection stays alive.
3. **Cancellation.** You cannot abort what you cannot observe. Interrupt requires a
   stream.

On Opus 5, single requests on hard agentic tasks can legitimately run for many minutes,
because thinking is on and effort is high. Plan for turns that take minutes, not seconds:
that shapes your timeouts, your progress UI, and whether your CLI can even be
synchronous.

---

## 3.2 The event shapes

You'll consume six event types. The exact wire format is in the docs; what matters is
the structure:

```
message_start          metadata, initial usage
  content_block_start  a block begins; carries its type (text / thinking / tool_use)
  content_block_delta  incremental content for block[index]
  content_block_stop   block[index] complete
message_delta          stop_reason, final output token count
message_stop
```

Two things to notice:

- **Blocks arrive by index, interleaved.** You're assembling an array, not a string.
  Your renderer needs to know that block 0 might be `thinking`, block 1 `text`, block 2
  `tool_use`. Don't assume order or count.
- **Delta types differ per block type.** `text_delta` carries `text`, `thinking_delta`
  carries `thinking`, and `input_json_delta` carries a *fragment of a JSON string* for a
  `tool_use` block's input. That last one is the interesting one.

### Partial tool input

Tool inputs stream as raw JSON fragments, which means mid-stream you're holding
`{"path": "src/ind` — not parseable. Two ways to use this:

- **Just accumulate and parse at `content_block_stop`.** Correct, simple, and what you
  should do first. The SDK helpers do this for you and hand you the parsed object.
- **Parse incrementally** with a partial-JSON parser so you can render "Reading
  src/index.ts..." before the block completes. Nice UX, real complexity. Note that by
  default the API buffers tool inputs somewhat; `eager_input_streaming: true` on the
  tool definition makes them stream more aggressively (this is a plain tool field, not
  a beta).

Do the simple version. Know that the other exists and why you'd want it.

### Use the SDK's accumulator

Every SDK has a helper that consumes the stream, accumulates the final `Message`, and
exposes both. In TypeScript: `client.messages.stream(...)`, `stream.on("text", ...)`
for deltas, and `await stream.finalMessage()` for the complete object. Do **not**
hand-roll a `new Promise()` around the event callbacks — the helper already handles
error, abort, and completion states, and getting those three right by hand is a
surprising amount of work.

Your loop's structure barely changes: stream for display, then `finalMessage()` to get
the object you append to `messages` and switch on. The invariants from stage 1 are
identical.

---

## 3.3 Cancellation, and the state problem it creates

The naive interrupt — `process.exit()` on Ctrl-C — is fine for a toy and useless for an
agent, because agents are worth interrupting *and then continuing*. "Stop, you're going
the wrong way, do this instead" is the most common interaction in real use.

So: pass an `AbortSignal` through the whole stack. The SDK request takes one. Your tool
executors take one (a `bash` child process should be killed, an in-flight HTTP request
aborted). `ctx` is the natural carrier.

Now the subtle part. **What does the message history look like after an interrupt?**
Three cases, and each needs a valid resulting array:

**Case A: interrupted during model generation, before any `tool_use`.**
You have a partial assistant message. Either discard it entirely (simplest, and what I'd
do first) or append the partial text. Do not append a partial `tool_use` block — you'd
have a `tool_use` with no `tool_result`, which is invariant 2 violated, and your next
request 400s.

**Case B: interrupted after `tool_use` blocks arrived, before you executed them.**
This is the trap. You've already got a complete assistant message with N `tool_use`
blocks. Invariant 2 says every one needs a result. So you must append a user message
with N `tool_result` blocks, each marked `is_error: true` with something like
"Interrupted by user before execution." Only then is the history valid.

**Case C: interrupted mid-tool-execution.**
Same as B, but some tools may have completed and had side effects. Return real results
for those, interruption errors for the rest. Note the honesty requirement: if a `write`
completed before the interrupt, the file *is* changed, and saying otherwise makes the
model's world model wrong.

The general rule, worth remembering as a sentence: **an interrupt must leave the
conversation in a state that is valid as the input to the next request.** Cancellation
is a state-machine problem, not a signal-handling problem. This is a good thing to be
able to say out loud.

### Interrupt-then-steer

Once B is handled correctly, the follow-up is easy and it's the payoff: after the
interruption results, append the user's new message and continue the same loop. The
model sees "I asked for three tools, they were interrupted, and the user said do this
instead." That's coherent, and it behaves well.

---

## 3.4 Retries and transient failure

The SDKs already retry 408, 409, 429, and 5xx with exponential backoff (default 2
retries), and they read the `retry-after` header on 429s. Do not write your own retry
loop on top without a reason — you'll stack backoffs and turn a 30-second blip into a
five-minute stall. Configure `maxRetries` instead.

What you *do* need to handle yourself:

- **Timeout units differ by SDK.** TypeScript is milliseconds, Python and Ruby are
  seconds. This is a classic self-inflicted outage.
- **Wall-clock can reach `timeout x (maxRetries + 1)`.** Size your outer deadline
  accordingly.
- **529 `overloaded_error`** is retryable but may need longer patience than the default.
- **Catch a chain of typed errors, not one broad class.** `NotFoundError` (bad model
  id — not retryable) and `RateLimitError` (retryable) demand opposite responses.
  Catching `APIError` once and retrying everything means you retry your own bugs.
- **Idempotency.** If a request times out after the model generated a response you
  never saw, retrying re-runs the turn. If that turn had side effects... it didn't,
  because side effects come from *your* executor, which didn't run. This is one of the
  few places the stateless API design makes life easier. But it does mean you pay twice.

---

## 3.5 Build it

1. Convert the loop to `client.messages.stream(...)` + `finalMessage()`. Render text
   deltas to stdout as they arrive.
2. Raise `max_tokens` to 64000 now that you're streaming. Note the difference in how
   the model behaves when it isn't budget-constrained.
3. Show tool calls as they're announced: `content_block_start` for a `tool_use` block
   gives you the tool name before the input finishes streaming. Render
   `[bash] running...` then the input once parsed.
4. Thread an `AbortController` through the API call, the loop, and every tool executor.
   Verify a `bash` child process actually dies on Ctrl-C (check `ps`).
5. Implement all three interrupt cases from 3.3. Prove case B by interrupting exactly
   between "tool requested" and "tool executed" — add a temporary 3-second sleep before
   dispatch to give yourself a window.
6. After an interrupt, prompt for a new user message and continue the same session.
7. Opt into visible reasoning: `thinking: { type: "adaptive", display: "summarized" }`,
   and render `thinking_delta` in a dim colour. Note that this is an opt-in — the
   default is `"omitted"`, which streams thinking blocks with empty text and looks
   like a long pause.

---

## 3.6 Break it on purpose

1. **Append a partial `tool_use` block** after an interrupt. Read the 400.
2. **Interrupt in case B and don't append interruption results.** Read the 400. Now
   you understand why cancellation is a state-machine problem.
3. **Don't propagate the abort signal to your `bash` executor.** Interrupt during a
   `sleep 60` and watch the process survive. Then check whether its output eventually
   arrives and confuses your renderer.
4. **Go back to non-streaming with `max_tokens: 64000`** and a prompt that produces a
   long answer. Observe the timeout (or, in Python, the SDK refusing outright).
5. **Set `maxRetries: 0`** and run enough concurrent requests to trigger a 429. Then set
   it back and watch the SDK absorb it silently.

---

## 3.7 Checkpoint

- [ ] Text streams token by token; tool calls announce before their input is complete.
- [ ] Ctrl-C during generation, during dispatch, and during tool execution all leave a
      history you can successfully send in the next request.
- [ ] After an interrupt you can type a new instruction and the same session continues
      coherently.
- [ ] A `bash` subprocess is actually killed on interrupt (verified with `ps`).
- [ ] `max_tokens: 64000` works without a timeout.

**Questions to be able to answer without looking:**

- Give three independent reasons streaming is required, not just nice.
- You interrupt after the model requested two tools but before you ran them. What
  exactly must you append to the message array, and why?
- How does tool input arrive over the stream, and what can't you do with it mid-flight?
- Why shouldn't you write your own retry wrapper around the SDK?

Next: [Stage 4 — Context engineering](04-context-engineering.md). This is the one that
matters most. Give it two sessions.

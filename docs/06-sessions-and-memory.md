# Stage 6 — Sessions & memory: state belongs on disk

**Milestone:** an append-only event log per session, `--resume` that reconstructs a
conversation and continues it, and a memory file the agent maintains across sessions.

**Time:** half a session. Mechanically straightforward; the interesting parts are the
log-as-source-of-truth idea and the difference between session state and memory.

---

## 6.1 Why an event log, not a message array dump

The naive approach is to serialise the `messages` array to JSON on exit. It works until
it doesn't:

- The process crashes mid-turn and you lose the whole session.
- You want to know *when* things happened, what they cost, which agent did what.
- You want to fork a session at turn 12 and try a different path.
- You want to replay a session against a modified harness to test a change.

The better model: an **append-only event log** is the source of truth, and the `messages`
array is a *projection* of it. This is just event sourcing, and it's the right shape
because a session genuinely is a sequence of events.

One JSONL file per session. Append after every event, flush immediately. Events:

```
{ t, type: "session_start",  model, cwd, systemPromptHash }
{ t, type: "user_message",   content }
{ t, type: "assistant",      content: [...blocks...], stopReason, usage }
{ t, type: "tool_call",      id, name, input }
{ t, type: "tool_result",    id, output, isError, durationMs }
{ t, type: "permission",     tool, decision, scope }
{ t, type: "compaction",     beforeTokens, afterTokens, summary }
{ t, type: "interrupt" }
{ t, type: "session_end",    reason }
```

Two properties to preserve deliberately:

- **Append-only.** Never rewrite history in the log, even when you compact. Compaction
  is itself an event; the projection applies it. This means you can always see what the
  agent originally saw, which is invaluable when debugging why it did something dumb.
- **The projection is pure.** `events -> messages` should be a function with no side
  effects. That's what makes replay and fork possible.

### Resume, and the validity trap

Replaying is where stage 1's invariants come back. Your projection must produce a
**valid** message array, which means:

- Every `tool_use` block has a matching `tool_result`. If the process died between
  requesting a tool and logging its result, the log has an orphan. Your projection must
  synthesise a `tool_result` with `is_error: true` ("interrupted") for it. This is
  exactly stage 3's case B, arriving from a different direction — and noticing that
  they're the same problem is the insight.
- Thinking blocks are reproduced byte-identically, signatures intact. So log the whole
  content array, not a reduction of it. If you logged only text, you cannot resume.
- The message array must start with a `user` message and end in a state the API accepts.

Test resume by `kill -9`-ing the process mid-tool-execution. That's the honest test; a
clean exit proves nothing.

### Fork

Once projection is pure, fork is `events.slice(0, n)` and a new session id. Cheap to
build, and genuinely useful: "try this again but tell it to use the existing helper
instead." Also the foundation of any eval harness that wants to test alternative paths
from a common prefix.

---

## 6.2 Session state vs memory

These get conflated. They're different things with different lifetimes:

| | Session state | Memory |
|---|---|---|
| Lifetime | One task | Across tasks, indefinitely |
| Contents | The full transcript | Distilled, durable facts |
| Size | Grows to 100Ks of tokens | Should stay small |
| Who writes it | Your harness, automatically | The agent, deliberately |
| Loaded into context | All of it (that's the conversation) | Selectively |

Memory is the "offload" move from stage 4 applied across sessions. Three ways to do it,
in increasing order of machinery:

**1. A file the agent reads and writes.** Simplest and surprisingly effective. A
`MEMORY.md` or a `.friday/memory/` directory. You mention it in the system prompt; the
agent uses `read`/`write`/`edit`, which it already has. No new mechanism at all. Start
here.

**2. The memory tool.** An Anthropic-defined client-side tool
(`{ type: "memory_20250818", name: "memory" }`) with a fixed command set (`view`,
`create`, `str_replace`, `insert`, `delete`, `rename`) operating on a `/memories`
directory. You implement the storage backend; the model already knows the interface.
Worth using because the model is trained on it, so you get better usage patterns for
free. **Every path is untrusted** — apply the stage-2 sandbox rigorously.

**3. A hosted memory store.** Managed persistence with versioning and audit trails.
Relevant if you're on a hosted platform; not something to build.

### What makes memory work or fail

The mechanism is easy. Getting *useful* memory is the hard part, and the lessons are
consistent:

- **One fact per file, with a summary line at the top.** Retrieval is by scanning
  summaries, so the summaries are the index.
- **Record the "why," not just the "what."** "Use `pnpm`, not `npm` — the workspace
  protocol breaks otherwise" is useful. "Use pnpm" is a rule the agent will violate the
  moment it seems inconvenient.
- **Don't store what's already discoverable.** Code structure, git history, and anything
  in the repo's own docs are not memory; they're a `read` away. Memory is for things
  that aren't written down anywhere.
- **Update, don't accumulate.** Duplicate near-identical notes are worse than none,
  because retrieval gets noisy. Instruct the agent to update an existing note.
- **Delete what turns out to be wrong.** Stale memory is actively harmful — it's
  confidently-asserted false context.
- **Never store secrets.** Memory is replayed verbatim into every future session that
  loads it. An API key written once is leaked into every subsequent context.

That list is also a decent answer to "how would you design memory for an agent?", which
is a common interview question with a lot of hand-waving in most answers.

---

## 6.3 Build it

1. `session/log.ts` — append-only JSONL writer, flushed per event. A session id and a
   directory (`.friday/sessions/<id>.jsonl`).
2. Emit every event type above from the places they occur.
3. `project(events) -> messages` as a pure function, including orphan-tool_use repair.
4. `--resume <id>` and `--resume-last`.
5. `--fork <id> --at <n>`.
6. A `sessions list` command showing id, first user message, turn count, token total,
   cost. You'll use this constantly.
7. Memory: option 1 first (a `MEMORY.md` the agent maintains, mentioned in the system
   prompt). Then, if you want the practice, implement the memory tool's six commands
   against a sandboxed directory.

---

## 6.4 Break it on purpose

1. **`kill -9` mid-tool-execution**, then resume. If you get a 400 about an unmatched
   `tool_use`, your projection doesn't repair orphans.
2. **Log only the text of assistant messages**, then resume a session that had thinking
   blocks. Note that you cannot reconstruct a valid request. This is why you log the
   whole content array.
3. **Rewrite history in the log** when compacting (mutate the file in place). Then try to
   debug why the agent made a decision three turns ago. Notice that you destroyed the
   evidence.
4. **Let memory accumulate without updating.** Run ten sessions on the same repo and see
   what the memory directory looks like. Then load it all into context and note the cost
   and the contradictions.
5. **Put a fake API key in a memory file**, then start a fresh session and ask the agent
   what it knows about the project. Watch it read the key back to you. Now you understand
   why the rule exists.

---

## 6.5 Checkpoint

- [ ] Sessions are logged append-only, and the log survives `kill -9`.
- [ ] `--resume` works after an ungraceful kill mid-tool-execution.
- [ ] Projection is a pure function you could call in a test.
- [ ] Fork works and you've used it to try an alternative path.
- [ ] The agent maintains a memory file across sessions, and you've seen it use a fact
      it learned in an earlier session.

**Questions to be able to answer without looking:**

- Why an event log rather than serialising the message array?
- What must the projection do that a naive replay wouldn't, and why?
- What's the difference between session state and memory?
- What makes a memory entry useful versus noise? Give four properties.
- Why should compaction be an event rather than a rewrite?

Next: [Stage 7 — MCP & extensibility](07-mcp-and-extensibility.md).

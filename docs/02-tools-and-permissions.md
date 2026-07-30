# Stage 2 — Tools & permissions: where product quality lives

**Milestone:** a tool registry with six tools (`bash`, `read`, `write`, `edit`, `glob`,
`grep`), schemas generated from `zod`, a permission gate that prompts before mutations,
and a path sandbox that can't be escaped.

**Time:** one session, possibly two. This is the stage with the most code and the least
novelty, so it's tempting to rush. The design questions in 2.1 are the interesting part.

---

## 2.1 The central design question: bash, or dedicated tools?

You already have `bash`. `bash` can do everything: read files, write files, search,
compile, deploy, delete your home directory. So why does Claude Code ship `Read`,
`Write`, `Edit`, `Glob`, and `Grep` as separate tools when `bash` subsumes all of them?

The answer is the best tool-design principle in this space, and it's a great interview
answer because most people haven't thought about it:

> **The harness can only gate, render, audit, or parallelise what it can see.**

A `bash` call gives your harness one opaque string. Every action has the same shape.
Your harness cannot tell `grep -r TODO .` from `curl -X POST https://prod/delete-all`.
Promoting an action to a dedicated tool gives you typed arguments and an
action-specific hook. Concretely, four things become possible:

**1. Security boundaries.** You can gate `write` behind confirmation while leaving
`read` automatic, because they're different tools with different names. You cannot
gate "the destructive half of bash" — you'd have to parse shell, and shell is
adversarially hard to parse (`$(...)`, `` ` ``, `&&`, `;`, aliases, `eval`).
Reversibility is the useful axis here: hard-to-reverse actions deserve their own tool.

**2. Invariant enforcement.** A dedicated `edit` tool can reject a write when the file
has changed since the model last read it. That's an invariant bash structurally cannot
enforce, and it prevents a whole class of clobbering bugs. More on this in 2.4.

**3. Rendering.** Some actions deserve UI. A dedicated `edit` can render a diff. A
dedicated "ask the user a question" tool can render a modal and block the loop until
answered. With bash, you can render a command string, and that's all.

**4. Scheduling.** You can mark `read`, `glob`, and `grep` as concurrency-safe and run
them in parallel. When the same actions arrive as bash, you must serialise everything,
because you can't distinguish a parallel-safe `grep` from a `git push`.

**The rule of thumb:** start with bash for breadth. Promote an action to a dedicated
tool when you need to gate it, render it, audit it, enforce an invariant on it, or
parallelise it. Keep bash around forever as the escape hatch — the long tail of "run
the test suite," "check git status," "install a package" is not worth 40 dedicated tools.

The mirror-image failure is a bloated tool surface. Too many tools degrades selection
accuracy and burns context on schemas the model will never use. If you find yourself at
20+ tools, the answer isn't better descriptions, it's either consolidation or
progressive disclosure (stage 7).

---

## 2.2 The registry

Your tool abstraction needs more than the three fields the API takes. The API sees
`{ name, description, input_schema }`; your harness needs metadata to make the decisions
above. Something in this shape:

```
Tool = {
  name         string
  description  string                     // what the API sees
  schema       ZodSchema                  // -> input_schema via zod-to-json-schema
  readOnly     boolean                    // safe to auto-approve? safe to parallelise?
  execute      (input, ctx) => Promise<Result>
}
```

Design notes, each of which you'll thank yourself for:

- **Validate the input before executing.** The model *usually* produces schema-valid
  input, and `strict: true` on the tool definition tightens that further, but "usually"
  is not an invariant you want your `rm` path depending on. Parse with zod; on failure
  return a `tool_result` with `is_error: true` and the validation message. The model
  reads it and corrects itself, which is a much nicer recovery than a crash.
- **`ctx` carries per-run state**, not globals: the working directory root, the
  permission gate, the abort signal (stage 3), the set of files read so far (2.4), the
  session log (stage 6). Pass it explicitly. Global mutable state will bite you the
  moment you add subagents in stage 5.
- **The return value needs more than a string.** At minimum: the content to send back,
  an `isError` flag, and optionally a separate "what to display to the user" (a rendered
  diff is not what you want in the model's context, and vice versa). Splitting
  *model-facing* from *user-facing* output early saves a refactor.
- **Never build the API request by hand-writing schemas twice.** One source of truth
  (the zod schema), generate the JSON Schema from it. Duplicated schemas drift, and the
  drift manifests as the model passing arguments your executor doesn't expect.

**On JSON Schema support:** not everything is supported. Basic types, `enum`, `const`,
`anyOf`, `allOf`, `$ref`, and string formats work. Numeric bounds (`minimum`,
`maximum`), string length bounds, and recursive schemas are **not** enforced.
`additionalProperties: false` is required on objects if you use `strict: true`. So
validate client-side regardless; treat the schema as a hint to the model, not a
guarantee.

---

## 2.3 The six tools, and the non-obvious bits of each

**`read(path, offset?, limit?)`**
The interesting question is what you do about size. A 40,000-line file will blow your
context budget on one call. Options: truncate with an explicit marker and a hint about
how to page (`offset`/`limit`), or refuse and tell the model to grep first. Pick one and
be consistent. Note what you do about images and binary files — reading a PNG as UTF-8
puts garbage in the context, and garbage in the context is expensive garbage.
Also: prepend line numbers. It makes `edit` and error messages dramatically better.

**`write(path, content)`**
Full-file overwrite. Should require prior confirmation. Should tell you whether it
created or overwrote. Consider refusing to overwrite a file that hasn't been read (2.4).

**`edit(path, old_string, new_string)`**
The important design decision: **require `old_string` to match exactly once.** Zero
matches is an error ("string not found"). Two or more matches is also an error
("string appears 3 times; include more surrounding context to disambiguate"). This
sounds pedantic and is the single highest-value invariant in a coding agent, because
the alternative — replace-first-occurrence — silently edits the wrong line and the
model has no way to notice.

**`glob(pattern)`** and **`grep(pattern, path?, glob?)`**
Both read-only, both parallel-safe. `grep` should cap the number of results and say so
when it caps. Use `ripgrep` if it's available; the performance difference matters when
the model is impatient and greps five times.

**`bash(command, timeout?)`**
Keep it. Add a timeout, an output cap, and a note in the description about what it
should *not* be used for ("Do not use for reading or editing files; use `read` and
`edit`, which are faster and safer"). Without that note the model will keep using
`cat` and `sed`, because `bash` is generic and generic tools attract calls.

### Error messages are a first-class interface

You are writing errors for a reader who will act on them autonomously. The difference
between these two is several wasted turns:

```
Error: ENOENT: no such file or directory, open 'src/config.ts'
```
```
File not found: src/config.ts
Nearby files in src/: config.json, configure.ts, index.ts
```

The second one gets corrected on the next turn. The first one triggers three `ls` calls.
Every error message is an opportunity to hand the model the information it needs to
recover in one step. Budget real effort here — it's cheaper than the tokens it saves.

---

## 2.4 The staleness invariant

Scenario: the model reads `config.ts` at turn 3. At turn 11 it issues an
`edit` against content it saw eight turns ago. Meanwhile you edited the file in your
editor, or a previous `bash` command reformatted it. The edit either fails confusingly
or succeeds against a stale mental model and destroys your change.

The fix: track, in `ctx`, a map of `path -> mtime-or-hash at last read`. `edit` and
`write` refuse if the file has changed since, with an error that says so and instructs
the model to re-read. Claude Code enforces exactly this, and it's why `Edit` errors with
"File has been modified since read."

Two things to notice:

- This is only possible because `edit` is a dedicated tool. `bash sed -i` cannot be
  gated this way. That's principle 2 from 2.1 made concrete.
- The invariant is enforced by your harness, not requested in the prompt. **Prefer
  mechanism to instruction.** Anything you can enforce in code, enforce in code —
  prompts are probabilistic, executors are not. This is a good sentence to have ready
  in an interview.

---

## 2.5 The permission gate

Where does it sit? Between "the model requested a tool" and "you execute it," inside the
loop. Three outcomes:

```
allow  ->  execute silently
ask    ->  block, prompt the human, then execute or refuse
deny   ->  return a tool_result explaining the refusal
```

Design points that matter more than they look:

- **A denial is a `tool_result`, not an abort.** Return `is_error: true` with "The user
  declined to run this command." The model then adapts — asks why, proposes an
  alternative, or explains what it needed. If you throw instead, you've thrown away the
  turn and the model's plan.
- **Decisions need scope.** "Allow once," "allow this tool for the rest of the session,"
  and "always allow this tool for this project" are three different things, and the
  third needs to persist to disk. Rule-matching on tool name plus a pattern over the
  input (allow `bash` when the command starts with `git status`) is how real harnesses
  do it. Note the danger: pattern-matching over shell strings is exactly the parsing
  problem from 2.1, so keep the patterns conservative and prefix-anchored.
- **Default posture is a product decision, not a technical one.** Read-only tools
  auto-approve; mutations ask; network egress and destructive operations ask with a
  louder prompt. Getting this wrong in either direction kills the product: too many
  prompts and nobody uses it, too few and it deletes something.
- **Approval in one context does not extend to the next.** If the user approved
  `rm build/` an hour ago, that isn't consent for `rm src/`. Scope decisions narrowly.

### The path sandbox

Every path from the model is untrusted input. Before any filesystem operation:

1. Resolve the path to its canonical absolute form, following symlinks.
2. Check that the result is inside your project root.
3. Reject otherwise.

The bugs live in the details: `..` traversal, symlinks pointing outside the root,
absolute paths, URL-encoded traversal (`%2e%2e%2f`), and — the one that gets people —
doing the check on the *input string* rather than on the *resolved* path. Use your
language's real path resolution (`fs.realpath` / `path.resolve` then a prefix check
on the resolved value), never string manipulation.

Note that this sandbox is advisory: `bash` can trivially escape it, since you're handing
a string to a shell. That's fine and expected at this stage — real isolation is a
container, and that's stage 8. What you're building here is a guard against accidents
and confused-model behaviour, not against an adversary.

---

## 2.6 Build it

Order that keeps you unblocked:

1. `tools/registry.ts` — the `Tool` type, a `Registry` that holds tools and produces
   the API's `tools` array, and a `dispatch(toolUse, ctx)` that validates and executes.
2. Port `bash` into the registry. Your stage-1 loop should now go through it unchanged.
3. `read`, `glob`, `grep` (all read-only, all auto-approve).
4. `write`, `edit` (both gated, both staleness-checked).
5. `permissions.ts` — the gate, with the three outcomes and at least session-scoped
   "always allow."
6. Run read-only tools in parallel when the model requests several in one turn.
   `Promise.all` over the read-only subset; serialise the rest. Log the wall-clock
   difference — it's usually large enough to be satisfying.

Keep `src/minimal.ts` from stage 1 untouched. This is a new file.

### Prompts to try

1. `"add a --verbose flag to the CLI"` — should read, then edit. Watch whether it reads
   before editing without being told. (It will. Note that this is behaviour, not
   enforcement — hence 2.4.)
2. `"find every place we call console.log and replace it with a logger"` — multi-file,
   many edits, several permission prompts. This is where your gate's UX becomes real.
3. `"what does this project do?"` — should be glob + read + grep, ideally in parallel.
4. `"delete all the test files"` — say no at the prompt. Confirm the model handles the
   refusal gracefully rather than retrying in a loop.

---

## 2.7 Break it on purpose

1. **Escape the sandbox.** Ask the agent to read `../../../../etc/passwd`. Then
   `~/.ssh/id_rsa`. Then create a symlink inside your project pointing at `/etc` and
   read through it. If any of those succeed, your check is on the string, not the
   resolved path.
2. **Make `edit` replace the first match instead of requiring uniqueness.** Ask it to
   change a common string like `const` or `return`. Watch it silently edit the wrong
   line. Revert to the strict version and feel the relief.
3. **Remove the staleness check.** Start a task, and while the model is working, edit
   the same file in your editor. Observe the clobber.
4. **Throw on permission denial** instead of returning `is_error`. Note how the run
   dies rather than adapting.
5. **Give `read` no size cap** and read a large generated file (a lockfile, a bundle).
   Watch your token count and cost for that single turn. This is the first taste of
   stage 4.
6. **Write a deliberately vague tool description** for `grep` — just "Searches." — and
   see how often the model reaches for `bash grep` instead. Then fix the description
   and re-run. The description *is* the interface.

---

## 2.8 Checkpoint

- [ ] All six tools work through one registry, with zod-generated schemas.
- [ ] Read-only tools execute in parallel; you can show the wall-clock difference.
- [ ] You cannot read outside the project root via `..`, absolute path, or symlink.
- [ ] `edit` refuses non-unique matches and refuses stale writes, with error messages
      that tell the model how to recover.
- [ ] Denying a permission prompt produces a graceful model response, not a crash.
- [ ] You can articulate, in two sentences, why `edit` exists when `bash sed` would do.

**Questions to be able to answer without looking:**

- What can a dedicated tool do that a bash command cannot, and why?
- How do you prevent the model clobbering a file that changed under it?
- Why is a permission denial returned as a tool result rather than raised as an error?
- What's the correct way to validate a model-supplied file path, and what's the common
  wrong way?
- What goes wrong at 40 tools, and what are the two fixes?

Next: [Stage 3 — Streaming & control](03-streaming-and-control.md).

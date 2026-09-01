---
name: delegate-to-codex
description: Use when the user explicitly asks for repository work to be handed to Codex: "delegate this to codex", "have codex do it", or the slash command. Covers choosing the model, writing the handoff, running `codex exec`, reviewing the diff that comes back, and committing it. Never reach for it unasked: work you could do directly, do directly. Not for use from inside Codex unless the user asks for nested delegation.
---

# Delegate to Codex

The orchestrator owns task selection, decomposition, final review, Git history, and user communication. Codex owns inspecting the repository, choosing implementation details, editing its bounded scope, and running validation.

**Only delegate when the user asked you to.** Large, tedious, cross-cutting, or precisely specified is not a trigger. Once they have asked, the delegation covers the whole piece of work under discussion, including follow-up runs and the commits that finish it. Do not re-ask per run.

## Decide whether to delegate

Delegate work needing sustained repository context: debugging, multi-file changes, migrations, refactors, test-backed implementation, substantial investigation. Do it yourself when it is a trivial edit, a one-command lookup, or smaller than the overhead of composing, running, and reviewing a second agent call.

Split into independently reviewable outcomes, never file-sized fragments. Keep a vertical change together when its schema, implementation, tests, and docs must agree. A single coherent run beats several context-starved calls, so split only for two reasons: a run killed for running too long, or a task wide enough to be worth a cold repository re-read per run.

When you do split:

- Run the pieces sequentially in one worktree, committing between each. Concurrent runs need a separate worktree per process; never two Codex processes in one tree.
- Give every prompt a boundary sentence naming the files the *other* runs own.
- Name the intermediate state the split invents, mark it temporary in the code, and forbid tests that assert it. The next run has to delete such a test, and a deleted assertion is indistinguishable from a weakened one under review.

## Select the model and effort

Respect an explicit user choice. Otherwise:

- `gpt-5.6-luna`: cost-sensitive, high-volume work with an unusually clear path and success check. Avoid where a subtle mistake would be expensive.
- `gpt-5.6-terra`: mechanical, well-scoped implementation. Routine test additions, narrow refactors, config changes, dependency bumps, ordinary features with clear acceptance criteria.
- `gpt-5.6-sol`: anything needing judgment. Architecture, ambiguous behavior, unfamiliar or cross-cutting code, hard debugging, security and data-safety work, migrations, performance, or a change whose correct shape must be discovered. Reach for it freely.
- `gpt-5.5` and `gpt-5.4`: previous generations. On request, for compatibility, or for a workflow already evaluated on them.

`gpt-5.6` aliases `gpt-5.6-sol`; use the explicit name so routing is obvious. If the selected model is not enabled for the local account, report that and fall back only where permitted.

A detailed plan does not make a task mechanical. Judge by the cost of a subtle mistake, not by how precisely you wrote the prompt. Judge breadth separately from difficulty: a wide task invites the cheaper model to scope itself down without saying so, so move up a model or narrow the run.

**Default to `low` effort, Sol included.** Sol at `low` and at `medium` are hard to tell apart in the diff; the model carries a judgment-heavy task, not the reasoning tokens on top of it. Raise to `medium` for genuinely hard runs: a shape that has to be discovered, a bug with no reproduction yet, a migration holding several invariants true at once.

**`high` is not yours to spend.** Ask the user first, name the part that needs it, and run at `medium` if they decline or there is no one to ask. A user who asked for `high` has already authorized it. Otherwise do not ask about effort per call. Pass model and effort explicitly, and never swap a requested model without saying so.

## Establish the workspace

1. Resolve the exact repository or worktree path, and read the repo's instructions and release policy.
2. Snapshot `git status --short --branch`; record pre-existing edits and untracked files.
3. Define one bounded outcome, its acceptance criteria, and who owns commit and push.
4. Identify the destructive, external, privileged, or deploy-triggering actions that stay outside Codex's authority. Separate those from steps the repo's documented workflow requires, such as a translation or codegen script that calls a paid API: authorize those in the prompt or plan to run them yourself, or Codex stops and asks.
5. If the change adds to an enumerable set (MCP tools, routes, nav entries, status values, error codes), find every assertion that counts or lists it before writing the prompt and name those files as authorized to update. `grep -rn "toHaveLength([0-9]" <test dirs>`, plus a grep for the count as a numeral and spelled out in prose, finds most.
6. Accessible names are an enumerable set too. Reusing a component puts its labels on screen twice, and `getAllByRole` then fails by ambiguity rather than by behavior. Grep the locale catalog first, and say the new instance needs its own strings while the original's stay byte-identical.

A clean worktree makes "every change in the diff is Codex's" true, which is what lets you review by diff alone. It survives only while nothing else writes to the tree. If the scope overlaps dirty files, say so in the prompt or isolate the work.

Let Codex load the repo's own `AGENTS.md`, user config, and skills. Do not paste them in, and do not use `--ignore-user-config` or `--ignore-rules` outside reproducibility testing.

## Write the handoff prompt

Give Codex the intent and everything that affects correctness, then get out of the way on implementation detail.

Separate three kinds of input explicitly, because Codex weighs them differently:

- **Verified external facts.** A spec revision, a changed API, a fetched doc. Label them authoritative and say *"do not correct these from memory"*, or a model restores the field name it remembers. Declare here too any in-repo idiom that pattern-matches to a famous bug, such as money kept in whole units and rounded like cents.
- **Settled user decisions.** Say "do not revisit these."
- **Your design sketch.** Label it *shape suggestions*, subordinate to what the repo actually does.

**Pass conclusions and coordinates, not transcripts.** Codex re-reads the repo anyway, so a pasted survey is paid for twice and is stale by the time it is read. Carry only the exact `path/file.ts:line`, the helper that already solves the adjacent problem, and the invariant written down nowhere.

Template, omitting empty sections:

```markdown
You are the implementing agent for one bounded repository task.

Goal: <The finished behavior or artifact, not activities. Include the acceptance bar.>

Context: <Requirements, issue details, observed errors, evidence.>

Verified external facts, AUTHORITATIVE. Do not correct these from memory: <Spec/API facts checked this session, with the exact names and values.>

Decisions already made, do not revisit: <Settled choices, so they are not relitigated.>

Design, shape suggestions only. Follow the repository where it differs: <Layering, key signatures, sketches.>

Scope and constraints:
- Work only in the current repository/worktree.
- Read and follow all applicable AGENTS.md and repository instructions.
- Inspect the existing implementation and follow established local patterns.
- Preserve pre-existing user changes: <paths, or "the worktree is clean">.
- Do not modify <out-of-scope areas>; report the need instead.
- Do not commit, push, deploy, publish, or perform external writes.
- Resolve ordinary implementation details autonomously from repository evidence.
- If a material ambiguity would change public behavior, data safety, or scope, stop and explain rather than guessing.

Tripwires, existing invariants that must not break:
- Do NOT edit existing tests to make them pass. A failing existing assertion means the implementation is wrong.
- <Named fragile contracts: exact strings, counts, serialized shapes, orderings.>

Acceptance criteria:
- <Observable criterion 1>

Validation:
- Run <specific commands>. Report each command, its result, and anything not run.

Return a concise summary of the implementation, files changed, validation results, and remaining risks or questions. If you run out of room, stop and say plainly what you did not build. A truthful partial report is more useful than a tidy one.
```

Keep that last sentence. It converts a silent shortfall into a review item. Do not shrink the summary further; delegation's tokens go into reading the diff, and a tighter boundary saves more than terser prose.

**Naming the tripwires pays off more than anything else in the prompt, and it needs a carve-out or it backfires.** "Do not edit existing tests to make them pass", plus the exact fragile contracts (a serialized shape, a count stated in prose, a deliberate ordering), stops a suite going green because the assertions moved. When your change legitimately alters an expectation, say so and say where: extending a list or updating a count to describe deliberate new behavior is correct, deleting a case or relaxing `toEqual` into `toContain` is not.

For a signature or type migration rather than a behavior change, ask for existing tests to be adapted with a local shim named after what it replaces, so the assertions stay byte-identical. A rewritten assertion and a weakened one are the same diff shape.

**Read your own constraints for self-contradiction before sending.** A prompt that forbids the only way to do what it demands costs a full round trip. Check each "do not" against each "must".

Ask for diagnosis only when diagnosis was requested; ask for implementation and validation when a change was requested. State each constraint once. Do not add "think step by step" or generic coding advice.

## Invoke Codex

Verified against codex-cli 0.147.0:

```bash
codex exec \
  -C /absolute/path/to/repository \
  -m gpt-5.6-sol \
  -c 'model_reasoning_effort="low"' \
  --approve-for-me \
  -o /unique/path/last-message.md \
  - < /unique/path/prompt.md
```

Send the prompt via stdin from a uniquely named file. Never interpolate user-controlled text into the shell.

- **`--approve-for-me` cannot be combined with `--sandbox`.** It already implies the workspace-write sandbox, and passing both fails argument parsing before any work happens. Use `-s read-only` without `--approve-for-me` for pure inspection.
- **Options belong to `codex exec`, so they must come before any subcommand.** `codex exec -C repo --approve-for-me -o out.md resume --last -` works. The same flags after `resume` die with `error: unexpected argument '--approve-for-me' found`, silently and looking like success.
- `--add-dir /exact/path` only when another writable directory is needed. `--json` when a program needs event data or a thread ID, paired with `-o`. Omit `--ephemeral` when a follow-up may resume the session. `--skip-git-repo-check` only for a verified non-repository workspace.
- Never `--dangerously-bypass-approvals-and-sandbox` unless the user explicitly authorizes it inside an already-isolated environment. `--full-auto` is deprecated.

If a flag is rejected, read `codex exec --help`. Do not guess replacement safety flags.

## Run it and confirm it actually ran

Long runs exceed foreground tool timeouts. Prefer background execution, writing the exit status where nothing can overwrite it:

```bash
{ codex exec ... > out.log 2> err.log; echo $? > exit.code; }
```

**Never append `; echo "EXIT=$?"` or any trailing command to the invocation**, and never read a pipeline's exit code. The trailing command's own success becomes the status, so a failed run reports 0, and `pnpm test 2>&1 | tail -40` reports `tail`'s status. Trust the runner's summary line.

A CLI argument error is silent and looks exactly like success: exit 2, empty stdout, no `-o` file, an unchanged worktree. Before reviewing anything, confirm `exit.code` is `0`, the `-o` file exists and is non-empty, and `git status` differs from your snapshot unless the task was read-only.

Codex echoes the entire prompt to stderr before it starts, then everything it reads and thinks, so the file runs to tens of kilobytes. Never `cat` it. Use `tail -c 2000`, and read the `-o` file for the result.

For concurrent runs: a distinct handle, log, result path, and thread ID per task; a separate worktree per editing process; wait on every process and check every exit status; never `resume --last`.

Treat nonzero exit as failed delegation, distinguishing a tool or argument failure from an incomplete code change. **A killed run is a different case**, and writes no exit-code file at all. Inspect before re-running: `git status` against your snapshot, `wc -l` the new files for truncation, the tail of stderr. A clean worktree means nothing is lost, so re-run narrower. A worktree of complete files means the kill landed during the summary and the work may be finished, so validate what is on disk and close small gaps yourself.

## Review

Codex's summary is a claim, not evidence. Verify independently:

1. Diff `git status` against the pre-run snapshot; note files it touched that you did not scope.
2. Read the full diff of the source changes.
3. Check that existing tests were not weakened: `git diff -U0 <test paths> | grep -E '^-[^-]'`. Deletions should be pure refactors, never removed assertions. Read the paired `+` line before judging; a replaced expectation and a deleted one are one character apart.
4. Re-run the type check, linter, and test suite yourself. Compare the test count to before.
5. Check acceptance criteria, edge cases, error handling, security and data implications, and consistency with local patterns.
6. Confirm nothing was committed, pushed, deployed, or published.

When auditing coverage, grep for the behavior (a distinctive identifier, an error code, a header name), not for `it(`. Table-driven tests written with `it.each` hide their cases from a title-only search.

**A green suite is Codex grading its own work.** It wrote those cases against the implementation it had just written, so they pass by construction. Read them for two things: the input the implementation does not handle, and any test that primes by hand the state the real wiring should produce. When a test sets up the integration point instead of exercising it, write the one that goes through the real path.

**When it stops and offers you options, check the repository before picking one.** The halt is the behavior you want, but the options are the ones it can see, and each may be a change where the answer is an existing convention. Grep for how the nearest sibling feature solves the same problem, and reply with that.

To correct, resume the same session with concrete evidence. Options precede the subcommand:

```bash
codex exec -C /absolute/path/to/repository --approve-for-me -o /unique/path/fix.md \
  resume <SESSION_ID> - < /unique/path/findings.md
```

Send findings via stdin from a file for anything longer than a sentence. Use `resume --last` only with exactly one recent session in that directory and no concurrency, and do not repeat model, effort, or sandbox overrides on resume unless changing them. If the same material problem survives two focused attempts, reconsider the prompt, the model, the boundary, or whether to do it yourself.

## Commit, push, and report

Git history stays under orchestrator control, so tell Codex not to commit or push. Validate before staging, then stage by name, never with `git add -A`. Re-list the changed files right before committing and compare them against the set you reviewed; another process in the same tree can add to them between the two steps. Where a pre-commit hook stashes unstaged changes, as lint-staged does, skip it with `--no-verify` while another process is live in the tree, and only once its checks have passed on the full tree.

Commit only when requested or clearly authorized; "implement X" is not authorization to commit X. Push only on explicit request or when the workflow requires it, checking first whether it triggers CI, deployment, or publication. Prefer atomic commits per stable outcome.

Report by leading with the verified outcome, not with the fact that another agent was used: what changed, what validation you ran yourself and its result, commit and push state, and material caveats or skipped checks. Mark which claims you verified and which are Codex's. Never report success solely because `codex exec` exited zero.

## Propose improvements to this skill

Once the work is reported, say what the session taught about delegating and offer it as a concrete edit. Do that unprompted, including when the session went well, and never edit this file unasked.

**The bar is whether it would have changed what you did**: a surprise that cost a round trip, a defect that survived a green suite, a boundary or prompt technique that visibly worked or failed, a model or effort choice wrong in a way you can diagnose. Not a data point confirming advice already here, and not a lesson you did not actually learn. Anything true only of the repository you worked in belongs in *that* repo's `AGENTS.md`.

**Fold in; do not append.** Every delegation reads this file start to finish, so its length is a cost paid on every run. Put a lesson inside an existing paragraph. When a session contradicts what is here, propose deleting that advice rather than qualifying it into vagueness. Commit the skill separately from the work it came out of.

## Avoid recursive delegation

When this skill is loaded inside Codex itself, do not invoke another Codex process unless the user explicitly asks for another layer. Work with the tools already available.

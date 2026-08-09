---
name: delegate-to-codex
description: Invoke Codex CLI as a subordinate implementation or investigation agent from another agent. Use when an outer agent should decompose substantial repository work, choose a Codex model and reasoning effort, construct a high-quality bounded prompt, run `codex exec`, monitor it, review its changes, request corrections, and optionally commit or push after verification. Do not use for ordinary direct work inside Codex or to recursively spawn Codex unless the user explicitly requests nested delegation.
---

# Delegate to Codex

The orchestrator owns task selection, decomposition, final review, Git history, and user
communication. Codex owns inspecting the repository, choosing implementation details,
editing its bounded scope, and running validation.

## Decide whether to delegate

Delegate work that benefits from sustained repository context: debugging, multi-file
changes, migrations, refactors, test-backed implementation, substantial investigation.

Do it directly when it is a trivial edit, a one-command lookup, or smaller than the
overhead of composing, running, and reviewing a second agent call.

Split a large request into independently reviewable outcomes, not file-sized fragments.
Keep a vertical change together when its schema, implementation, tests, and docs must
agree. Run dependent tasks sequentially. Run independent tasks concurrently only in
separate worktrees; never let two Codex processes edit one working tree.

A single coherent run beats several context-starved calls. Do not split merely because the
user said "split the task."

## Select the model and effort

Respect an explicit user choice. Otherwise:

- `gpt-5.6-luna` — cost-sensitive, high-volume work where the path and success check are
  unusually clear: repetitive edits, simple transformations, bulk test-case generation,
  or many small independent tasks. Avoid it when a subtle mistake would be expensive.
- `gpt-5.6-terra` — the default for mechanical and well-scoped implementation work:
  routine test additions, narrow refactors, repetitive config changes, dependency bumps,
  and ordinary feature changes with clear acceptance criteria. It balances capability and
  cost better than Luna when repository interpretation still matters.
- `gpt-5.6-sol` — anything needing judgment: architecture, ambiguous behavior, unfamiliar
  or cross-cutting code, hard debugging, security or data-safety work, migrations,
  performance analysis, or a change whose correct shape must be discovered.
- `gpt-5.5` — a strong previous-generation model; use on request, for a known compatibility
  reason, or when an existing evaluated workflow is pinned to it.
- `gpt-5.4` — use only on request, for compatibility, or for an existing workflow already
  evaluated on it. Prefer the current GPT-5.6 family for new delegation.

`gpt-5.6` is an alias for `gpt-5.6-sol`; prefer the explicit `gpt-5.6-sol` name in
automation so routing is obvious. Do not assume every API-catalog model is enabled for the
local Codex account. If the selected model is unavailable, report that fact and choose a
fallback only when the user or enclosing workflow permits it.

A detailed plan does **not** make a task mechanical. Judge by the cost of a subtle mistake,
not by how precisely you wrote the prompt.

Default to `medium` effort. Do not ask the user to pick effort per call. Model capability
and reasoning effort are separate choices; pass both explicitly so nothing is inherited
silently. Do not silently swap an explicitly requested model.

## Establish the workspace

Before invoking:

1. Resolve the exact repository or worktree path.
2. Read the repo's instructions and release/deployment policy.
3. Snapshot `git status --short --branch`; record pre-existing edits and untracked files.
4. Define one bounded outcome, its acceptance criteria, and who owns commit and push.
5. Identify destructive, external, privileged, or deploy-triggering actions that stay
   outside Codex's authority.

A clean worktree is worth creating before you start — it makes "every change in the diff is
Codex's" true, which is what lets you review by diff alone. If the scope overlaps dirty
files, say so in the prompt or isolate the work.

Let Codex load the repo's own `AGENTS.md`, user config, and skills. Do not paste them into
the prompt. Do not use `--ignore-user-config` or `--ignore-rules` outside reproducibility
testing.

## Write the handoff prompt

Give Codex the intent and everything that affects correctness. Then get out of the way on
implementation detail — it should inspect the repo and match local patterns.

**Separate three kinds of input explicitly**, because Codex weighs them differently:

- **Verified external facts** — a spec revision, a changed API, a fetched doc. Codex cannot
  derive these from the repo and **its training data may contradict them**. Label them
  authoritative and say *"do not correct these from memory."* Without that line, a model
  will quietly "fix" a correct new field name back to the one it remembers.
- **Settled user decisions** — say "do not revisit these."
- **Your design sketch** — label it as *shape suggestions*, subordinate to what the repo
  actually does.

Template — omit empty sections:

```markdown
You are the implementing agent for one bounded repository task.

Goal:
<The finished behavior or artifact, not activities. Include the acceptance bar.>

Context:
<Requirements, issue details, observed errors, evidence.>

Verified external facts — AUTHORITATIVE, do not correct from memory:
<Spec/API facts checked this session, with the exact names and values.>

Decisions already made — do not revisit:
<Settled choices, so they are not relitigated.>

Design (shape suggestions — follow the repository where it differs):
<Layering, key signatures, sketches.>

Scope and constraints:
- Work only in the current repository/worktree.
- Read and follow all applicable AGENTS.md and repository instructions.
- Inspect the existing implementation and follow established local patterns.
- Preserve pre-existing user changes: <paths, or "the worktree is clean">.
- Do not modify <out-of-scope areas>; report the need instead.
- Do not commit, push, deploy, publish, or perform external writes.
- Resolve ordinary implementation details autonomously from repository evidence.
- If a material ambiguity would change public behavior, data safety, or scope, stop and
  explain rather than guessing.

Tripwires — existing invariants that must not break:
- Do NOT edit existing tests to make them pass. A failing existing assertion means the
  implementation is wrong.
- <Named fragile contracts: exact strings, counts, serialized shapes, orderings.>

Acceptance criteria:
- <Observable criterion 1>

Validation:
- Run <specific commands>. Report each command, its result, and anything not run.

Return a concise summary of the implementation, files changed, validation results, and
remaining risks or questions.
```

Naming the tripwires is the highest-leverage part of the prompt. "Do not edit existing
tests to make them pass" plus a list of the exact fragile contracts (a serialized payload
shape, a count stated in prose, a deliberate ordering) prevents the failure mode where the
suite goes green because the assertions moved.

Ask for diagnosis only when diagnosis was requested; ask for implementation and validation
when a change was requested. Do not request a plan-only pass unless the plan is the
deliverable. State each constraint once. Do not add "think step by step" or generic coding
advice.

## Invoke Codex

Verified against codex-cli 0.147.0:

```bash
codex exec \
  -C /absolute/path/to/repository \
  -m gpt-5.6-sol \
  -c 'model_reasoning_effort="medium"' \
  --approve-for-me \
  -o /unique/path/last-message.md \
  - < /unique/path/prompt.md
```

**`--approve-for-me` cannot be combined with `--sandbox`.** It already implies the
workspace-write sandbox; passing both fails argument parsing before any work happens. Use
`--approve-for-me` for unattended implementation; use `-s read-only` (without
`--approve-for-me`) for pure inspection.

Send the prompt via stdin from a uniquely named file written with your normal file tool.
Never interpolate user-controlled text into the shell.

Other flags:

- `--add-dir /exact/path` only when another writable directory is genuinely needed.
- `--json` when a program needs event data or an unambiguous thread ID; pair with `-o`.
- Omit `--ephemeral` when a follow-up may need to resume the session.
- Never `--dangerously-bypass-approvals-and-sandbox` unless the user explicitly authorizes
  it inside an already-isolated environment. `--full-auto` is deprecated.
- `--skip-git-repo-check` only for a verified non-repository workspace.

If a flag is rejected, read `codex exec --help` and adapt to the installed version. Do not
guess replacement safety flags.

## Run it and confirm it actually ran

Long runs exceed foreground tool timeouts. Prefer background execution with the exit status
written where nothing can overwrite it:

```bash
{ codex exec ... > out.log 2> err.log; echo $? > exit.code; }
```

**Never append `; echo "EXIT=$?"` or any trailing command to the invocation.** The trailing
command's own success becomes the command's exit status, so a failed run is reported to
your harness as exit 0.

A CLI argument error is silent and looks exactly like success: **exit 2, empty stdout, no
`-o` file, and an unchanged worktree** — indistinguishable at a glance from "ran and
decided to change nothing." Before reviewing anything, confirm all three:

1. `exit.code` is `0`;
2. the `-o` file exists and is non-empty;
3. `git status` differs from your snapshot (or the task was genuinely read-only).

Codex echoes the **entire prompt to stderr** before it starts working. An early `tail` of
stderr showing your own prompt text means "started", not "progress" — do not read it as
output.

For concurrent runs: a distinct handle, log, result path, and thread ID per task; a
separate worktree per editing process; wait on every process and check every exit status;
never `resume --last`.

Treat nonzero exit as failed delegation. Distinguish a tool/argument failure from an
incomplete code change, and fix the cause before retrying.

## Review

Codex's summary is a claim, not evidence. Verify independently:

1. Diff `git status` against the pre-run snapshot; note files it touched that you did not
   scope.
2. Read the full diff of the source changes.
3. **Check that existing tests were not weakened**, cheaply:
   `git diff -U0 <test paths> | grep -E '^-[^-]'` — deletions should be pure refactors
   (a signature gaining an optional parameter, an import line), never removed assertions.
4. Re-run the type check, linter, and test suite **yourself**. Compare the test count to
   before; it should have gone up by roughly the number of cases you asked for.
5. Check acceptance criteria, edge cases, error handling, security and data implications,
   and consistency with local patterns.
6. Confirm nothing was committed, pushed, deployed, or published.

When auditing test coverage, grep for the *behavior* (a distinctive identifier, an error
code, a header name), not for `it(` — table-driven tests (`it.each`) hide their cases from
a title-only search and will make you conclude coverage is missing when it is not.

To correct, resume the same session with concrete evidence:

```bash
codex exec -C /absolute/path/to/repository resume <SESSION_ID> \
  "Address these review findings, rerun the relevant checks, and report: <findings>"
```

Use `resume --last` only with exactly one recent session in that directory and no
concurrency. Do not repeat model/effort/sandbox overrides on resume unless changing them.

Limit correction loops. If the same material problem survives two focused attempts, stop
and reconsider the prompt, the model, the task boundary, or whether to do it yourself.

## Commit and push

Git history stays under orchestrator control:

- Tell Codex not to commit or push.
- Review and validate before staging; stage only files belonging to the bounded task.
- Write a commit message describing the verified outcome.
- Commit only when requested or clearly authorized. "Implement X" is not authorization to
  commit X.
- Push only on explicit request or when the enclosing workflow requires it. Check first
  whether the push triggers CI, deployment, or publication.

Prefer atomic commits per stable outcome; one coherent commit for tightly coupled parts of
a single feature.

## Report

Lead with the verified outcome, not with the fact that another agent was used. Include what
changed, what validation you ran yourself and its result, commit/push state if any, and
material caveats or skipped checks.

Mark clearly which claims you verified and which are Codex's. Never report success solely
because `codex exec` exited zero.

## Avoid recursive delegation

When this skill is loaded inside Codex itself, do not invoke another Codex process unless
the user explicitly asks for another layer. Work with the tools already available.

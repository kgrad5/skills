# Unit of Work: Standard Process for Claude Code

**Purpose:** A repeatable process for taking one task from "picked up" to "merged" using Claude Code, GitHub Issues, and the `gh` CLI. Follow it every time you start a unit of work.

**A unit of work is:** one task, small enough for a single Claude session to plan and implement, with a verification check that can pass or fail. If a task doesn't fit that description, split it before starting.

---

## Step 0 — Decide whether you need the full process

Not every change deserves planning. Planning pays off when you're uncertain about the approach, the change touches multiple files, or you're unfamiliar with the code.

**Skip to a direct ask** (no plan mode, no issue ceremony) when you could describe the diff in one sentence: fixing a typo, adding a log line, renaming a variable. Just tell Claude what to do, verify, and commit.

One exception. A one-line change can still be architecture-significant — pinning a runtime, adding a dependency, making a foreign key nullable, changing a default other code will come to rely on. **Size is not the test; reversibility is.** Such a change earns a decision-log entry even though it does not earn an issue: write the entry in the same commit at `**Status:** Accepted · <YYYY-MM-DD>`, with the issue field *omitted* rather than filled — everywhere in this process, fields you do not have are omitted, not filled. Without this, the decision gate at plan time is trivially evaded by not opening an issue.

**Use the full process below** for everything else.

---

## One-Time Setup (per repository)

Do this once per repo, not per task.

1. **CLAUDE.md exists and is short.** Run `/init` to generate one, then prune. Include only: build/test/lint commands, style rules that differ from defaults, repo etiquette (branch naming, PR conventions), and gotchas. If a line wouldn't cause mistakes when removed, delete it. Bloated CLAUDE.md files cause Claude to ignore your actual instructions. Check it into git.
2. **`gh` CLI installed and authenticated.** Run `gh auth login`. Claude knows how to use `gh` for issues and PRs; without it, unauthenticated API calls hit rate limits.
3. **A check exists and runs by command.** Tests, build, lint — something that returns pass/fail. If your repo has no runnable check, fix that before anything else. This process does not work without one.
4. **Permissions configured.** Run `/permissions` and allowlist the commands you trust (test runner, linter, `git commit`). Optionally enable `/sandbox` for OS-level isolation so routine commands run without prompts.

---

## Where Tasks Come From (once per feature)

Individual units of work come from a spec. For any feature bigger than one task, do this before entering the loop:

### 1. Interview → spec

Start a fresh Claude Code session and run this prompt (fill in the bracket):

```
I want to build [brief description]. Interview me in detail using the
AskUserQuestion tool.

Ask about technical implementation, UI/UX, edge cases, concerns, and
tradeoffs. Don't ask obvious questions, dig into the hard parts I might
not have considered.

Keep interviewing until we've covered everything, then write a complete
spec to specs/<slug>.md.
```

A good spec is self-contained: it names the files and interfaces involved, states what is **out of scope**, is accompanied by one file in `arch/` for every decision the interview settled and the options it closed off, and ends with an end-to-end verification step that proves the feature works. Edit the spec until it meets that bar. Commit it, with its `arch/` files.

### 2. Spec → GitHub Issues

In the same session (or a fresh one with the spec referenced):

```
Read @specs/<slug>.md. Break it into an ordered list of tasks, each independently
implementable and verifiable. For each task, create a GitHub issue with
`gh issue create` containing:

- Goal (one paragraph)
- Files likely involved
- Out of scope (explicit)
- Verification: a concrete check that proves it's done
- Spec: the specs/ file this task came from

Label them all `feature-name` and number them in dependency order.
Show me the list before creating anything.
```

**The rule that makes this work:** every issue must carry a check that can pass or fail. "curl returns 429 after the 10th request" — not "rate limiting works." If you can't write the check, the task isn't defined yet.

Do **not** generate intermediate documents between the spec and the issues (PRDs, milestone docs, epic breakdowns). Each generative layer *re-derives* content that already exists somewhere else, so it drifts from intent and goes stale the moment implementation deviates. Spec + `arch/` + issues is the whole stack.

The architecture decision record is the one exception, and not because of where it sits. It is append-only, it restates nothing — each entry is the sole record of a decision and the alternatives it closed off — and it cannot go stale, because superseding an entry is itself an entry. It lives in `arch/`, one file per decision, in every repo — spec or no spec. A superseded entry moves into `arch/archive/` rather than being deleted: it keeps its number and its wording forever, and agents ignore it except to count numbers and to resolve a constraint on an issue older than the supersede. That is what keeps the log both complete and small enough to read. If your org needs milestones and epics for coordination, keep them in your tracker for humans — they are not inputs to the agent.

---

## The Loop (per unit of work)

Repeat these ten steps for every task. One task per session.

### 1. Start clean

Open a fresh session in plan mode:

```bash
claude --permission-mode plan
```

Or, in an existing terminal: run `/clear`, then press `Shift+Tab` until the status bar shows `⏸ plan mode on`. Never carry a previous task's context into a new one — a clean session with a good prompt outperforms a long session with accumulated corrections.

### 2. Load the task

```
Run `gh issue view <NUMBER>` and read the spec its Spec: line names. Restate the goal,
the out-of-scope list, and the verification check in your own words.
```

If Claude's restatement is wrong, the issue is ambiguous. Fix the issue, not the conversation.

### 3. Explore

Still in plan mode — Claude reads files and answers questions without making changes:

```
Read the files this task likely touches and the patterns they follow.
[Point at specifics: "read /src/auth and understand how we handle
sessions" / "look at how existing widgets are implemented —
HotDogWidget.php is a good example."]
```

Point at real files and existing patterns. The more precise the context, the fewer corrections later. For large codebases, tell Claude to use subagents for the research so file contents don't flood your main context.

### 4. Plan

```
Create a detailed implementation plan for this issue: files to change,
the approach, the test plan, side effects (migrations, env vars, new
dependencies), and risks. Stay inside the issue's scope.
```

Press `Ctrl+G` to open the plan in your editor and edit it directly. Review against this checklist before approving:

- [ ] Every consequential decision is explicit (no "handle appropriately")
- [ ] The listed files match what you'd expect, plus the decision log when the plan records an entry
- [ ] The plan ends with the issue's verification check
- [ ] Nothing outside the issue's scope has crept in — recording a decision this plan makes is inside scope
- [ ] Test plan covers the edge cases named in the issue
- [ ] The plan names the decision-log entry it adds, or says in as many words that it adds none. An entry is earned when either holds: the decision makes a sentence in the spec wrong or incomplete, or it chose between options whose loser would have produced a different schema, file layout, dependency, or wire format

Push back in plain English until it's right. Arguing with a paragraph is cheaper than reverting a diff.

### 5. Approve — and choose the leash

Approving the plan exits plan mode and sets the permission mode for execution. Pick deliberately:

| Situation | Choose |
|---|---|
| Unfamiliar code, risky change, prod-adjacent | **Review each edit** manually |
| Routine change in code you know | **Accept edits**, review the diff after |
| Well-scoped task with a strong verification check | **Auto mode** — background safety checks, no per-tool prompts |

### 6. Implement

```
Implement the approved plan. Write tests for [the specific behavior],
run the test suite, and fix any failures. Address root causes,
don't suppress errors.
```

Watch loosely, not line-by-line. If Claude veers off: press `Esc` to interrupt and redirect immediately. Small corrections early beat big corrections late.

### 7. Verify with evidence

```
Run the verification check from the issue. Show me the exact command
you ran and its full output.
```

Require evidence — test output, the command and its result, a screenshot — not an assertion of success. Reviewing evidence is faster than re-running the check yourself. If the check can't be run, the task is not done; there is no exception to this.

### 8. Adversarial review

Get a fresh set of eyes that isn't biased by the reasoning that produced the change. Either run the bundled `/code-review` skill (correctness review of the current diff in a fresh subagent), or check the diff against the plan:

```
Use a subagent to review the diff against the approved plan. Check that
every requirement is implemented, the listed edge cases have tests, and
nothing outside the task's scope changed. Report gaps, not style
preferences.
```

Fix gaps that affect correctness or stated requirements. Ignore the rest — a reviewer told to find gaps will always find some, and chasing every finding produces over-engineering.

### 9. Commit and PR

```
Commit with a descriptive message and open a PR with `gh pr create`.
Include "Closes #<NUMBER>" in the PR description and summarize the
verification evidence in the PR body. State which decision-log entries
this diff adds, or that it adds none.
```

`Closes #N` auto-closes the issue on merge, which keeps the backlog truthful without manual bookkeeping.

### 10. Close out

Merge per your team's review rules. Then run `/clear` (or close the terminal) before touching the next task. Done means: check passed with evidence, review gaps addressed, PR merged, issue closed, and every architecture decision the work settled is in the log on the default branch.

---

## Course-Correction Rules

- **`Esc`** — interrupt Claude mid-action; context is preserved, redirect immediately.
- **Two-strike rule** — if you've corrected Claude twice on the same issue, stop. The context is polluted with failed approaches. Run `/clear` and restart with a better prompt that incorporates what you learned.
- **`Esc Esc` or `/rewind`** — restore conversation and/or code to a previous checkpoint. Use it to try risky approaches cheaply: attempt, and if it fails, rewind. (Checkpoints only cover Claude's file edits, not Bash side effects — it's not a git replacement.)
- **`/compact <instructions>`** — if a legitimately long task approaches context limits, compact with focus instructions rather than letting auto-compaction guess what matters.

---

## Anti-Patterns (and the fix)

| Anti-pattern | Fix |
|---|---|
| Kitchen-sink session: multiple unrelated tasks in one context | `/clear` between tasks; one issue per session |
| Correcting the same mistake repeatedly | Two-strike rule: `/clear` + better prompt |
| Planning the whole project upfront through document cascades (PRD → milestones → epics → tasks) | Spec + issues only; plan per-task in plan mode, with real code in context |
| Skipping verification because it "looks done" | No check, no done. Write the check into the issue before work starts |
| Bloated CLAUDE.md or spec that Claude ignores | Prune ruthlessly; move rules that must always fire into hooks |
| Unscoped "investigate X" that floods context | Scope the investigation or delegate it to subagents |

---

## Quick Reference

| Action | Command / Key |
|---|---|
| Start session in plan mode | `claude --permission-mode plan` |
| Toggle plan mode in session | `Shift+Tab` (until `⏸ plan mode on`) |
| Edit the plan in your editor | `Ctrl+G` |
| Interrupt Claude | `Esc` |
| Rewind to checkpoint | `Esc Esc` or `/rewind` |
| Reset context | `/clear` |
| Focused compaction | `/compact <instructions>` |
| Manage tool permissions | `/permissions` |
| Correctness review of diff | `/code-review` |
| View a task | `gh issue view <n>` |
| Create a task | `gh issue create` |
| Open a PR | `gh pr create` (body: `Closes #<n>`) |

**Source:** Claude Code best practices — https://code.claude.com/docs/en/best-practices

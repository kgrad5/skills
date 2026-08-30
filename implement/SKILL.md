---
name: implement
description: Creates the issue's branch and implements a previously approved plan under a strict scope guard, refusing to touch any file the plan does not list and refusing to run at all if no approved plan can be found. This skill should be used when the user says "implement the plan", "implement issue 42", "build it", or "start implementing" after a plan has been approved.
argument-hint: [issue-number]
---

# implement

Guarantees that the work done matches the plan that was approved — and that scope never expands without the user saying so.

## Preconditions

Check and report; never assume.

- `gh auth status` succeeds.
- A test command is discoverable: check `CLAUDE.md`, then `package.json` scripts, `Makefile`, `pyproject.toml`, `Cargo.toml`, `go.mod`. If none is found, **ask**. Never invent a test command.

## Inputs

Arguments given: $ARGUMENTS

Resolve the issue number in this order. Stop at the first that yields one.

1. The first argument, if it matches `^#?[0-9]+$`.
2. `git branch --show-current` — the first number matching `(?:^|[/-])([0-9]+)(?:-|$)`. Covers `42-rate-limit`, `feat/42-rate-limit`, `kgrad/issue-42`.
3. `gh pr view --json body -q .body` — the number in `[Cc]loses #([0-9]+)`.
4. Ask the user. Never scan the issue list for a likely candidate, and never guess.

## Procedure

1. **Retrieve the approved plan**, in this order:
   - This conversation, if `/issue-to-task` ran in it
   - `gh issue view <N> --json comments` — the **last** comment containing `<!-- unit-of-work:approved-plan -->`
   - The harness plan file, if the session reported one exists

   **If no plan is found anywhere, stop and refuse.** Say: no approved plan for issue `<N>`; run `/issue-to-task <N>` first. Implementing from the issue alone is exactly the unplanned work this process exists to prevent, and it is the failure mode most likely to look like success.

   If the plan came from conversation and no marked comment exists on the issue yet, post one now (see `/issue-to-task` step 10) before starting work.

2. **Read the issue** for its out-of-scope list and verification check: `gh issue view <N>`. Read `./SPEC.md` if present; if absent, say "no SPEC.md; working from the issue alone" and continue.

3. **Create the branch**: `gh issue develop <N> --checkout`. This names the branch for the issue and registers it as GitHub's linked branch, so every later skill can infer the number from the branch. If that fails, fall back to `git switch -c <N>-<short-slug>`. If already on the issue's branch, stay.

4. **Restate the plan's file list** and state plainly that anything not on it is off-limits for this task.

5. **Write the tests the plan named, first.** Run them. Watch them fail for the right reason. A test that passes before the implementation exists is testing nothing.

6. **Implement.** Then run the full suite.

7. **Fix failures at the root.** Never suppress an error, loosen an assertion, add a retry around a real bug, or mark a test skipped to get to green.

8. **If the work needs a file outside the plan's list — stop.** Name the file, say why it is needed, and ask. Never expand scope silently. The user may approve it, redirect you, or decide the plan was wrong; all three are better than finding out in the diff.

## Hard rules

- **No approved plan, no implementation.** This is the same rule as `/verify`'s "no check, no done."
- **The plan's file list is the boundary.** Off-list work stops and asks.
- **Root causes only.** Suppressing an error is not fixing it.
- **Two-strike rule.** If the user has corrected the same mistake twice, stop implementing. Say that the context is now polluted with failed approaches and that continuing will compound it. Recommend `/clear`, then `/issue-to-task <N>` again with what was learned written into the issue or the plan. Two corrections is a signal about the context, not about the next attempt.
- **Nothing outside this issue.** Unrelated dead code, adjacent style, a bug you noticed in passing — mention them, do not fix them. They are other units of work.

## Output

Report:
- The issue number and the branch created or reused.
- Where the plan came from (conversation, issue comment, or plan file).
- The plan's file list, and the files actually changed. Call out any divergence explicitly.
- The test command run and whether the suite passes.
- Anything you stopped and asked about.

## Next step

Run `/verify` to produce evidence against the issue's check. Do not claim the task is done on the strength of a passing suite — the issue's own check is the gate.

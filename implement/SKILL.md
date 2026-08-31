---
name: implement
description: Creates the issue's branch and implements a previously approved plan under a strict scope guard, writes any ADR the plan drafted into arch/, refusing to touch any file the plan does not list and refusing to run at all if no approved plan can be found. This skill should be used when the user says "implement the plan", "implement issue 42", "build it", or "start implementing" after a plan has been approved.
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

2. **Read the issue** for its out-of-scope list and verification check: `gh issue view <N>`, and the spec its `Spec:` line names. A repo predating this layout may carry a root `SPEC.md`; use it and say so. If it names none, say "no spec named; working from the issue alone" and continue.

3. **Create the branch**: `gh issue develop <N> --checkout`. This names the branch for the issue and registers it as GitHub's linked branch, so every later skill can infer the number from the branch. If that fails, fall back to `git switch -c <N>-<short-slug>`. If already on the issue's branch, stay.

4. **Restate the plan's file list** and state plainly that anything not on it is off-limits for this task. An `arch/` file is on that list whenever the plan's Architecture decisions section holds an entry; take the exact path from the plan.

5. **Write the tests the plan named, first.** Run them. Watch them fail for the right reason. A test that passes before the implementation exists is testing nothing.

6. **Implement.** Then run the full suite.

7. **Write the plan's drafted ADR into `arch/`.** Create the file at the path the plan named, copying the entry body verbatim and changing exactly two things: `**Status:** Proposed` becomes `**Status:** Accepted`, and the date becomes the day the plan was approved.

   **Only ever add files here. Never rename, renumber or delete one, and never rewrite one.** Setting a superseded entry's Status line to `Superseded by ADR-<m>` is the single permitted edit to a file already in `arch/`, and only when the approved plan drafted that supersede — its own words stay exactly as they were, because the reversal is the most useful thing in the record and a rewritten entry destroys it.

   If the plan's Architecture decisions section says `None`, the writing half of this step is a no-op — but the archiving half below still runs, because a backlog is not this unit of work's doing and waiting for a plan that mentions it is waiting forever. If the plan has **no such section at all** — one approved before this step existed — do not infer its contents: treat any decision the work makes as unplanned and stop, per step 9. An entry left in the issue comment is lost: the comment is a draft, the file is the record.

   **Then archive what the log has retired.** `git mv` every file in `arch/` whose Status line reads `Superseded by ADR-<n>` into `arch/archive/`, keeping its filename byte for byte. Two kinds of file move here and both are in scope:
   - **The entry this unit of work just superseded.** The move is the second half of the supersede, so the plan's file list should already name it.
   - **Any entry an earlier unit of work superseded and left in place.** This sweep is the one file move the scope guard permits without asking. It adds no decision, alters no wording, and drains a backlog that empties no other way: nothing ever supersedes an already-superseded entry a second time, so an entry missed once stays missed.

   Four constraints, of which the third is the one that bites:
   - **Move, never copy or delete.** `git mv`, so history follows the file. The append-only promise holds in the sense that matters — no entry is ever destroyed, and the archive is part of the record rather than a wastebasket.
   - **Never touch the contents of a file you move.** Its Status line already names the entry that retired it.
   - **Grep for the filename first, and leave the file where it is if anything references it by path** — a test allowlist, a lint exemption, a docs link, an issue body. Report what you found instead of moving it. A sweep that turns a green suite red is not bookkeeping, and this step runs at the moment a reviewer is least expecting a file to move.
   - **Report every path you moved**, in your output and in the commit message. A move nobody announces reads as a deletion in review.

8. **Fix failures at the root.** Never suppress an error, loosen an assertion, add a retry around a real bug, or mark a test skipped to get to green.

9. **If the work needs a file outside the plan's list — stop.** Name the file, say why it is needed, and ask. Never expand scope silently. The user may approve it, redirect you, or decide the plan was wrong; all three are better than finding out in the diff.

   **A decision is not a file.** If the work settles something architecture-significant that the plan did not draft — even when an `arch/` file is already on the plan's list — stop and ask in exactly the same way. The scope guard counts files, so a second, unplanned decision walks straight through it and lands in the log wearing the authority of a plan that never considered it.

## Hard rules

- **No approved plan, no implementation.** This is the same rule as `/verify`'s "no check, no done."
- **The plan's file list is the boundary.** Off-list work stops and asks — and so does an off-plan *decision*, even when it touches only listed files. Step 7's archive sweep is the single exception, and it is one because it moves files without deciding anything; a sweep that would have to edit a file, or that something references by path, stops and asks like everything else.
- **Root causes only.** Suppressing an error is not fixing it.
- **Two-strike rule.** If the user has corrected the same mistake twice, stop implementing. Say that the context is now polluted with failed approaches and that continuing will compound it. Recommend `/clear`, then `/issue-to-task <N>` again with what was learned written into the issue or the plan. Two corrections is a signal about the context, not about the next attempt.
- **Nothing outside this issue.** Unrelated dead code, adjacent style, a bug you noticed in passing — mention them, do not fix them. They are other units of work.
- **Never write an ADR the plan did not draft.** The plan is what the user approved, and approval is what makes an entry `Accepted`. An entry written here that nobody approved is a decision smuggled into the record.

## Output

Report:
- The issue number and the branch created or reused.
- Where the plan came from (conversation, issue comment, or plan file).
- The plan's file list, and the files actually changed. Call out any divergence explicitly.
- The `arch/` files written, by path. Say nothing here when the plan drafted none — but a plan that drafted one and a step 7 that wrote nothing must never both pass in silence.
- The entries moved into `arch/archive/`, by path, and for each whether it was this unit of work's own supersede or a sweep of one left behind. Name any entry you declined to move and what referenced it. Omit the line entirely when nothing moved and nothing was blocked.
- The test command run and whether the suite passes.
- Anything you stopped and asked about.

## Next step

Run `/verify` to produce evidence against the issue's check. Do not claim the task is done on the strength of a passing suite — the issue's own check is the gate.

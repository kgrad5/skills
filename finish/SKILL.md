---
name: finish
description: Closes out a unit of work after the pull request was merged by hand in the GitHub UI — verifies the merge actually happened, confirms the issue closed and closes it explicitly if it did not, deletes the branch locally and remotely, and states the definition of done with evidence. This skill should be used when the user says "finish", "finish up", "I merged it", "clean up the branch", "close it out", or "wrap up issue 42".
argument-hint: [issue-number]
---

# finish

Guarantees that a merged pull request is genuinely finished: the issue closed rather than assumed closed, the branch gone, and the definition of done stated with evidence.

This is the tail `/ship` deliberately leaves off. `/ship` stops at the PR so the merge decision happens in the GitHub UI, where the diff is readable. `/finish` picks up afterwards. It is normally run in a fresh session, minutes or days later, with no memory of the work — that is expected; the branch, the PR, and the issue hold everything needed.

## Preconditions

Check and report; never assume.

- `gh auth status` succeeds.
- `gh repo view --json defaultBranchRef` resolves the base branch. Use that value; never hardcode `main` or `master`.

## Inputs

Arguments given: $ARGUMENTS

Resolve the issue number in this order. Stop at the first that yields one.

1. The first argument, if it matches `^#?[0-9]+$`.
2. `git branch --show-current` — the first number matching `(?:^|[/-])([0-9]+)(?:-|$)`. Covers `42-rate-limit`, `feat/42-rate-limit`, `kgrad/issue-42`.
3. `gh pr view --json body -q .body` — the number in `[Cc]loses #([0-9]+)`.
4. Ask the user. Never scan the issue list for a likely candidate, and never guess.

Then find the PR: `gh pr list --state all --search "Closes #<N> in:body" --json number,state,headRefName`. If more than one matches, ask which.

## Procedure

1. **Refuse unless the PR actually merged.**
   ```bash
   gh pr view <PR> --json state,mergedAt,mergeCommit,baseRefName,headRefName
   ```
   If `state` is not `MERGED`, **stop and change nothing.** Say whether it is `OPEN` (merge it first) or `CLOSED` without merging (the work was abandoned; deleting the branch would destroy it). This is the one way this skill can lose work, so it is the first gate.

2. **Refuse if the working tree is dirty.** `git status --porcelain`. Uncommitted changes on a branch about to be deleted are gone for good. Name the files and stop. Untracked files that are ignored by intent (a local `.env`, an editor directory) are worth naming but are not a reason to stop — say which you saw and let the user decide.

3. **Confirm the issue actually closed:**
   ```bash
   gh issue view <N> --json state,title
   ```
   If it is still `OPEN`, close it with `gh issue close <N>` and say plainly why the auto-close did not fire — usually a base branch that was not the repository's default, or a PR body that never carried `Closes #N`. `Closes #N` is a convention, not a guarantee, and this is the step that catches it.

4. **Return to the base branch and update it:**
   ```bash
   git switch <default-branch> && git pull --ff-only && git fetch --prune
   ```
   `--prune` clears the stale remote-tracking ref if GitHub already deleted the branch on merge.

5. **Delete the local branch.** Try `git branch -d <branch>` first. If it refuses because the PR was **squash-merged** — the branch's commits are not ancestors of the base, so git cannot prove it was merged — use `git branch -D <branch>`. Only ever reach for `-D` having confirmed `MERGED` in step 1.

6. **Confirm the remote branch is gone.** GitHub deletes it on merge only when the repository has automatic branch deletion enabled. If it survives, `git push origin --delete <branch>`.

7. **State the definition of done**, item by item, with the evidence for each:
   - The verification check passed — link the evidence comment on the issue, from `/verify` or `/ship`. If the issue carries no such comment, say so rather than implying it was checked.
   - The PR is merged, with the resulting commit on the default branch.
   - The issue is closed, and whether that happened automatically or had to be forced.
   - The branch is gone, locally and remotely.

   If any item cannot be confirmed, say which and why.

## Hard rules

- **No merge, no cleanup.** `state` must be `MERGED`. An unmerged PR's branch is the only copy of the work.
- **Never delete a branch with uncommitted changes on it.** Stop and name the files.
- **`-D` only after confirming `MERGED`.** Force-deleting on an assumption is how work disappears.
- **Verify the issue closed.** Never report it closed on the strength of `Closes #N` being present in the PR body.
- **Report the definition of done honestly.** If the issue has no verification evidence, say that — do not let a merged PR stand in for a check that passed.

## Output

Report:
- The issue and PR numbers, the branch, and the merge commit on the default branch.
- The issue's final state, and whether it closed automatically or had to be closed explicitly.
- What was deleted, locally and remotely.
- The four done criteria, each with its evidence.

## Next step

Run `/clear` before touching the next task, then `/issue-to-task <next-issue-number>`. Never carry a finished task's context into a new one.

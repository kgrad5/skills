---
name: finish
description: Closes out a unit of work after the pull request was merged by hand in the GitHub UI — verifies the merge actually happened, confirms the issue closed and closes it explicitly if it did not, confirms the arch/ files landed on the default branch, deletes the branch locally and remotely, and states the definition of done with evidence. This skill should be used when the user says "finish", "finish up", "I merged it", "clean up the branch", "close it out", or "wrap up issue 42".
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

Resolve the **issue** number in this order. Stop at the first that yields one.

1. **The first argument**, if it matches `^#?[0-9]+$` — but disambiguate it before trusting it. GitHub gives issues and pull requests **one shared number space**, and `gh issue view <n>` **succeeds on a PR number** (returning `state: MERGED`) rather than erroring. So a bare argument may be either.

   ```bash
   gh api repos/{owner}/{repo}/issues/<n> -q '{number, title, is_pr: (has("pull_request"))}'
   ```

   - `is_pr: false` → it is the issue. Use it as `<N>`.
   - `is_pr: true` → the user passed the **PR** number. Use it as the PR, and take `<N>` from that PR's body: `gh pr view <n> --json body -q .body`, then the number in `[Cc]loses #([0-9]+)`. Say which you did, so the user sees their argument was read as a PR.
   - HTTP 404 → neither an issue nor a PR with that number. Say so and ask; never fall through to the next rule, which would silently act on different work than the user named.

2. `git branch --show-current` — the first number matching `(?:^|[/-])([0-9]+)(?:-|$)`. Covers `42-rate-limit`, `feat/42-rate-limit`, `kgrad/issue-42`.
3. `gh pr view --json body -q .body` — the number in `[Cc]loses #([0-9]+)`.
4. Ask the user. Never scan the issue list for a likely candidate, and never guess.

Then, unless rule 1 already produced it, find the PR:

```bash
gh pr list --state all --search "Closes #<N> in:body" --json number,state,headRefName
```

If more than one matches, ask which. If none matches, say so and stop — an issue with no PR has nothing to finish.

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

5. **Confirm the architecture decisions landed — only if the PR recorded any.** If the PR body carries no `Decisions recorded:` line, there is nothing to confirm: skip to step 6 without comment. Otherwise: you are standing on the updated default branch, which is the only place this check means anything — the branch you came from always has the entry. Read the log from the branch, not the checkout, because a squash merge leaves none of the PR's commits behind:

   ```bash
   git ls-tree --name-only <default-branch> arch/
   ```

   Name the files belonging to this unit of work. If the PR body said `Decisions recorded: ADR-<n>` and the default branch does not hold it, say so plainly — that is a record that silently failed to land.

   **Then flag any retired entry still sitting in the top level, whether or not this PR recorded a decision.** `/implement` archives a superseded entry into `arch/archive/` as the second half of the supersede, and sweeps any it finds left behind. This is the backstop for what that sweep missed or declined:

   ```bash
   git grep -l '^\*\*Status:\*\* Superseded' <default-branch> -- arch/ ':!arch/archive/'
   ```

   Name every file it returns. **Do not move them** — see the hard rule below. Each one is a small unit of work for `/issue-to-task`, or a sign that something references the entry by path and `/implement` was right to leave it alone. Say which you think it is, and say nothing at all when the command returns nothing.

6. **Delete the local branch.** Try `git branch -d <branch>` first. If it refuses because the PR was **squash-merged** — the branch's commits are not ancestors of the base, so git cannot prove it was merged — use `git branch -D <branch>`. Only ever reach for `-D` having confirmed `MERGED` in step 1.

7. **Confirm the remote branch is gone.** GitHub deletes it on merge only when the repository has automatic branch deletion enabled. If it survives, `git push origin --delete <branch>`.

8. **State the definition of done**, item by item, with the evidence for each:
   - The verification check passed — link the evidence comment on the issue, which `/verify` posts. `/ship` puts its evidence in the PR body, not on the issue, so a unit of work that skipped `/verify` leaves the issue with none. If the issue carries no such comment, say so rather than implying it was checked.
   - The PR is merged, with the resulting commit on the default branch.
   - The issue is closed, and whether that happened automatically or had to be forced.
   - The branch is gone, locally and remotely.
   - The architecture decisions this work settled are in `arch/` on the default branch, by path. Omit this item when the work settled none.

   If any item cannot be confirmed, say which and why.

## Hard rules

- **No merge, no cleanup.** `state` must be `MERGED`. An unmerged PR's branch is the only copy of the work.
- **Never delete a branch with uncommitted changes on it.** Stop and name the files.
- **`-D` only after confirming `MERGED`.** Force-deleting on an assumption is how work disappears.
- **Verify the issue closed.** Never report it closed on the strength of `Closes #N` being present in the PR body.
- **Report the definition of done honestly.** If the issue has no verification evidence, say that — do not let a merged PR stand in for a check that passed.
- **Never write to `arch/` here, and never move within it.** Step 4 leaves you on the default branch with a clean tree, which makes both recording a missing decision and sweeping a retired one into `arch/archive/` look free. Either is an unreviewed commit straight to the default branch, outside any unit of work — and a move is the worse of the two, because it can break a build that references the entry by path with nothing between it and the default branch. Archiving belongs to `/implement`, where it rides a branch, a plan and a pull request. Report the gap; fixing it is the next `/issue-to-task`, not this skill's job.

## Output

Report:
- The issue and PR numbers, the branch, and the merge commit on the default branch.
- The issue's final state, and whether it closed automatically or had to be closed explicitly.
- What was deleted, locally and remotely.
- The five done criteria, each with its evidence.
- Any superseded entry still in the top level of `arch/`, by path, named as work for the next `/issue-to-task`. Omit the line when there is none.

## Next step

Run `/clear` before touching the next task, then `/issue-to-task <next-issue-number>`. Never carry a finished task's context into a new one.

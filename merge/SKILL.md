---
name: merge
description: Lands an approved pull request by checking that external review approved it and all status checks pass, squash-merging it, deleting the branch, and confirming the linked issue actually closed. This skill should be used when the user says "merge the PR", "land it", "the review is approved", or "merge issue 42".
argument-hint: [issue-number|pr-number]
---

# merge

Guarantees that nothing reaches the default branch without external approval and green checks — and that the issue is genuinely closed afterwards, not merely assumed to be.

## Preconditions

Check and report; never assume.

- `gh auth status` succeeds.
- `gh repo view --json defaultBranchRef` resolves the base branch. Use that value; never hardcode `main` or `master`.

## Inputs

Arguments given: $ARGUMENTS

Resolve the pull request in this order. Stop at the first that yields one.

1. The first argument, if it matches `^#?[0-9]+$` — try it as a PR number; if no such PR exists, treat it as an issue number and find the PR whose body contains `Closes #<n>`.
2. `gh pr view --json number` for the current branch.
3. `git branch --show-current` — the first number matching `(?:^|[/-])([0-9]+)(?:-|$)`, then the PR closing that issue.
4. Ask the user. Never merge a PR you inferred loosely.

Then resolve the issue number from the PR body's `[Cc]loses #([0-9]+)`.

This skill is normally run in a fresh session, hours or days after `/ship`, with no memory of the work. That is expected — the issue and the PR hold everything needed.

## Procedure

1. **Read the PR's state:**
   ```bash
   gh pr view <PR> --json number,title,baseRefName,headRefName,reviewDecision,mergeStateStatus,statusCheckRollup,body
   ```

2. **Gate on external review and checks. Refuse unless both hold:**
   - `reviewDecision` is `APPROVED`
   - Every entry in `statusCheckRollup` has concluded successfully

   If either fails, **merge nothing** and report exactly what blocks it: the review decision and who is requested or has requested changes, and each failing or pending check by name with its URL. Then stop. This gate holds identically for a human reviewer and for a bot — an unapproved PR does not land.

3. **Check the base branch.** If `baseRefName` is not the repository's default branch, say so — `Closes #N` will not auto-close the issue, and step 5 will have to do it explicitly.

4. **Confirm with the user, then merge:**
   ```bash
   gh pr merge <PR> --squash --delete-branch
   ```
   One commit per unit of work on the default branch, and the branch cleaned up on the remote. A merge is outward-facing and hard to undo; it gets an explicit yes first.

5. **Confirm the issue actually closed:**
   ```bash
   gh issue view <N> --json state,title
   ```
   If it is still `OPEN`, close it explicitly with `gh issue close <N>` and say plainly that the auto-close did not fire and why — usually a non-default base branch, or a PR body that never carried `Closes #N`.

6. **Clean up locally:**
   ```bash
   git switch <default-branch> && git pull
   ```
   Confirm the task branch is gone locally and remotely. If `--delete-branch` left a local copy, delete it.

7. **State the definition of done**, item by item, with the evidence for each:
   - The verification check passed, with evidence
   - Review gaps were addressed
   - The PR is merged
   - The issue is closed

   If any item cannot be confirmed, say which and why. Do not report a task done on three out of four.

## Hard rules

- **No approval, no merge.** `reviewDecision` must be `APPROVED`.
- **No green checks, no merge.** Every check concluded successfully, none pending.
- **Confirm with the user before merging.** Irreversible and outward-facing.
- **Never approve the PR yourself** to satisfy the gate. If it needs a reviewer, it needs a reviewer.
- **Verify the issue closed.** `Closes #N` is a convention, not a guarantee — check the state and close it explicitly when it did not fire.
- **Report the definition of done honestly.** Three of four items is not done.

## Output

Report:
- PR number, title, base and head branches.
- The review decision and every status check with its conclusion.
- What was merged, with the resulting commit on the default branch.
- The issue's final state, and whether it closed automatically or had to be closed explicitly.
- The four done criteria, each with its evidence.

## Next step

Run `/clear` before touching the next task, then `/issue-to-task <next-issue-number>`. Never carry a finished task's context into a new one.

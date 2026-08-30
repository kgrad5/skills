---
name: ship
description: Commits the work, runs a correctness review, checks the diff against the approved plan and the issue, re-runs the verification check, and opens a pull request containing Closes #N and fresh evidence. Stops there so the merge decision happens in the GitHub UI, where the diff is actually readable. This skill should be used when the user says "ship it", "open the PR", "raise a PR for this", or "commit and PR".
argument-hint: [issue-number] [--draft]
---

# ship

Guarantees that what reaches the pull request has been reviewed against the plan it came from, and that the evidence in the PR body was produced after the last change — not before it.

## Preconditions

Check and report; never assume.

- `gh auth status` succeeds.
- `gh repo view --json defaultBranchRef` resolves the base branch. Use that value everywhere; never hardcode `main` or `master`.
- The working tree contains changes to ship (`git status --porcelain`).

## Inputs

Arguments given: $ARGUMENTS

Resolve the issue number in this order. Stop at the first that yields one.

1. The first argument, if it matches `^#?[0-9]+$`.
2. `git branch --show-current` — the first number matching `(?:^|[/-])([0-9]+)(?:-|$)`. Covers `42-rate-limit`, `feat/42-rate-limit`, `kgrad/issue-42`.
3. `gh pr view --json body -q .body` — the number in `[Cc]loses #([0-9]+)`.
4. Ask the user. Never scan the issue list for a likely candidate, and never guess.

## Procedure

The order below is deliberate. Do not reorder it.

1. **Refuse if the current branch is the base branch.** Say so and stop. Nothing gets committed directly to the default branch. If `/implement` was skipped, run `gh issue develop <N> --checkout` first.

2. **Commit first**, with a descriptive message. This gives the reviewer a stable object and puts the work beyond the reach of `/rewind`, which does not cover Bash side effects.

3. **Delegate correctness review** to the bundled `/code-review` skill, scoped to this branch's diff.
   - `/code-review` forks and runs in the background by default. **If it returns an agent name or handle rather than findings, stop.** Report that the review is still running and that `/ship` should be re-run once it lands. Do not proceed past this gate on a handle — a review you did not read is not a review.
   - This pass and step 4 are the only scrutiny this change will get. There is no second human. Do not treat them as a formality to clear.

4. **Check plan conformance** — the thing `/code-review` cannot do, because it never sees the issue or the plan.
   - Retrieve the approved plan: this conversation, then `gh issue view <N> --json comments` for the last comment containing `<!-- unit-of-work:approved-plan -->`, then the harness plan file.
   - If no plan is retrievable, review the diff against **the issue alone**, and say so explicitly — in your report and in the PR body. Never claim conformance you did not check.
   - Apply **the same five criteria as `/issue-to-task`, to the diff instead of the plan**:
     - Every consequential decision in the plan is actually implemented — nothing "handled appropriately"
     - The files changed match the plan's file list
     - The issue's verification check is satisfied by the change
     - Nothing outside the issue's scope changed
     - The edge cases the issue names have tests
   - Use a subagent for this so it is not biased by the reasoning that produced the change.

5. **Fix the gaps that matter** — those affecting correctness or a stated requirement. Ignore style preferences. A reviewer told to find gaps will always find some, and chasing every finding produces over-engineering. Commit the fixes separately.

6. **Re-run the verification check now**, after the fixes. The evidence `/verify` produced earlier predates step 5 and is stale. Capture this run's exact command and full output — that is what goes in the PR body.

7. **Confirm with the user before opening the PR.** Show the title, base, the body you are about to post, and the commits. A pull request is outward-facing; it gets an explicit yes.

8. **Push the branch**: `git push -u origin HEAD`. Every commit from steps 2 and 5 is still local until this runs — `gh issue develop` created the remote branch at the base commit, not at your work. Without it `gh pr create` cannot determine a head repository and aborts with "you must first push the current branch to a remote".

9. **Open the PR**: `gh pr create --base <default-branch> --title <...> --body <...>`, plus `--draft` if the argument was given. The body must contain:
   - `Closes #<N>`
   - A short summary of the change
   - The verification evidence from step 6 — command and full output, in a fenced block
   - Any departure from the approved plan, called out rather than left in a commit message
   - A note if the plan-conformance check ran against the issue alone

   **If `--draft` was given, stop here.** A draft PR is a deliberate request to pause before landing.

10. **Check the base.** `Closes #N` only auto-closes the issue on merge into the repository's default branch. If the resolved base is anything else, say so plainly — the issue will not auto-close, and `/finish` will have to close it explicitly.

11. **Stop.** Report the PR URL and that the merge is now yours to make in the GitHub UI, where the diff is readable. Do not merge, do not wait.

## Hard rules

- **Never commit or PR from the default branch.**
- **Push before `gh pr create`.** It cannot resolve a head repository from an unpushed branch and will abort.
- **Never proceed past `/code-review` on an agent handle.** Findings or nothing.
- **Never claim plan conformance without the plan.** Say which artifact you reviewed against.
- **The evidence in the PR body is generated after the last fix.** Stale evidence in a PR is a false claim about tested code.
- **Confirm before `gh pr create`.** Outward-facing and hard to take back.
- **Do not merge.** The merge happens in the GitHub UI, then `/finish` cleans up.

## Output

Report:
- The branch, base branch, and commits made.
- `/code-review`'s findings, and which you fixed versus ignored, with reasons.
- The five plan-conformance verdicts, and what the plan came from.
- The verification command and its full output from step 6.
- The PR URL, and that the merge is yours to make.

## Next step

Merge the PR in the GitHub UI. Then run `/finish` — from a fresh session is fine; the branch and the issue hold everything needed.

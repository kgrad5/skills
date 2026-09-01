---
name: ship
description: Commits the work, runs a correctness review, checks the diff against the approved plan, the issue and arch/, re-runs the verification check, and opens a pull request containing Closes #N and fresh evidence. Stops there so the merge decision happens in the GitHub UI, where the diff is actually readable. This skill should be used when the user says "ship it", "open the PR", "raise a PR for this", or "commit and PR".
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
   - **The decision log is `arch/` at the repo root**, one file per decision. `arch/archive/` holds the entries later decisions retired; read it only to resolve a number, never to judge scope. The plan's file list names the exact paths it drafted. Read the issue's `Spec:` line for the spec this work descends from, and judge scope against that spec — not against a spec you guessed at.
   - Apply **the same six criteria as `/issue-to-task`, to the diff instead of the plan**:
     - Every consequential decision in the plan is actually implemented — nothing "handled appropriately"
     - The files changed match the plan's file list — with two exceptions. An `arch/` file may be **added** even where the plan omitted it, because criterion 6 firing *means the plan was incomplete*; say so explicitly, in the report and in the PR body. And a file may be **moved** from `arch/` to `arch/archive/` even where the plan's file list names only its old path, because `/implement` step 7 archives two kinds of entry: the one this unit of work just superseded, and any an earlier unit of work retired and left behind. Confirm its Status line reads `Superseded by ADR-<n>`, and that the move carried the contents over unchanged — the sole permitted difference is that Status line, on the entry this diff supersedes — then treat it as in scope
     - The issue's verification check is satisfied by the change
     - Nothing outside the issue's scope changed, except an `arch/` file recording a decision this diff makes
     - The edge cases the issue names have tests
     - **The diff adds the `arch/` file the plan named, and no decision in the diff is missing from `arch/`.** Check the artifact first: the plan's Architecture decisions section either names entries — each must appear in the diff — or holds the literal `None`, which the diff must bear out. The bar for a decision the plan never named is the same one `/issue-to-task` used: it makes a sentence in the issue's spec wrong or incomplete, or it chose between options whose loser would have produced a different schema, file layout, dependency, or wire format
   - Use a subagent for this so it is not biased by the reasoning that produced the change.

5. **Fix the gaps that matter** — those affecting correctness or a stated requirement. Ignore style preferences. A reviewer told to find gaps will always find some, and chasing every finding produces over-engineering. Commit the fixes separately.

   **A missing ADR is always a gap that matters, never a style preference.** Add `arch/<NNNN>-<kebab-slug>.md` in the format `/idea-to-spec` defines — `# ADR-<n>`, then Status, Context, Decision, Consequences, Alternatives rejected — numbered one past the highest file across `arch/` and `arch/archive/` together — a retired entry keeps its number, so a count that skips the archive reuses one. **Re-check the number against the default branch**: a plan drafted days ago may have claimed a number another branch has since landed. Renumber and rename rather than collide, then commit it with the other fixes. But an entry written here was never in an approved plan, so it cannot claim the approval that makes an entry `Accepted`: show its Context, Decision, Consequences and Alternatives to the user and get an explicit yes first — the same yes step 7 requires for the pull request itself.

6. **Re-run the verification check now**, after the fixes. The evidence `/verify` produced earlier predates step 5 and is stale. Capture this run's exact command and full output — that is what goes in the PR body.

7. **Confirm with the user before opening the PR.** Show the title, base, the body you are about to post, and the commits. A pull request is outward-facing; it gets an explicit yes.

8. **Push the branch**: `git push -u origin HEAD`. Every commit from steps 2 and 5 is still local until this runs — `gh issue develop` created the remote branch at the base commit, not at your work. Without it `gh pr create` cannot determine a head repository and aborts with "you must first push the current branch to a remote".

9. **Open the PR**: `gh pr create --base <default-branch> --title <...> --body <...>`, plus `--draft` if the argument was given. The body must contain:
   - `Closes #<N>`
   - A short summary of the change
   - The verification evidence from step 6 — command and full output, in a fenced block
   - Any departure from the approved plan, called out rather than left in a commit message
   - A note if the plan-conformance check ran against the issue alone
   - `Decisions recorded: ADR-<n>`, when this diff records any. The reader is about to merge in the GitHub UI; this is where they see the record keeping up with the code. Omit the line entirely when there are none — a PR body that announces its own emptiness teaches the reader to skip the line

   **If `--draft` was given, stop here.** A draft PR is a deliberate request to pause before landing.

10. **Check the base.** `Closes #N` only auto-closes the issue on merge into the repository's default branch. If the resolved base is anything else, say so plainly — the issue will not auto-close, and `/finish` will have to close it explicitly.

11. **Stop.** Report the PR URL and that the merge is now yours to make in the GitHub UI, where the diff is readable. Do not merge, do not wait.

## Hard rules

- **Never commit or PR from the default branch.**
- **Push before `gh pr create`.** It cannot resolve a head repository from an unpushed branch and will abort.
- **Never proceed past `/code-review` on an agent handle.** Findings or nothing.
- **Never claim plan conformance without the plan.** Say which artifact you reviewed against.
- **No ADR, no PR.** A diff that decides architecture and adds no `arch/` file does not become a pull request. Write the entry, get the user's yes, commit it — then open the PR.
- **Never stamp `Accepted` on text nobody approved.** An entry this skill authors gets the user's explicit yes before it is committed.
- **The evidence in the PR body is generated after the last fix.** Stale evidence in a PR is a false claim about tested code.
- **Confirm before `gh pr create`.** Outward-facing and hard to take back.
- **Do not merge.** The merge happens in the GitHub UI, then `/finish` cleans up.

## Output

Report:
- The branch, base branch, and commits made.
- `/code-review`'s findings, and which you fixed versus ignored, with reasons.
- The six plan-conformance verdicts, and what the plan came from.
- The `arch/` files this branch adds, by path, and whether any was authored here rather than in the plan. One line, omitted when there are none.
- The verification command and its full output from step 6.
- The PR URL, and that the merge is yours to make.

## Next step

Merge the PR in the GitHub UI. Then run `/finish` — from a fresh session is fine; the branch and the issue hold everything needed.

# Stage rules for /execute

Every stage the `/execute` orchestrator launches reads this file in full before its first command. It binds the stage alongside the stage's own skill. The orchestrator reads it too, at the start of every run.

## Gates

Each one is a point where a chained skill asks the user. Under `/execute` the stage answers it as written here, and leaves the answer where a human can audit it.

- **issue-to-task step 2 — open dependency.** Never proceed. Return `blocked`, naming it. If this fires, the graph was wrong.
- **issue-to-task step 4 — restatement.** Self-confirm. If writing the restatement shows the issue is ambiguous, edit the issue with `gh issue edit` so it says what it means, re-read it, and name the edit in the plan comment. Fix the issue, not the conversation.
- **issue-to-task — no verification check.** Write one into the issue under `## Verification`, derived from the issue's goal and the spec: a command and its expected output.
- **issue-to-task steps 5–9 — plan mode and approval.** There is no plan mode. Explore read-only, write the plan as text, gate it on the six criteria — fixing any that fail — and post it with the `<!-- unit-of-work:approved-plan -->` marker. The comment is the approval artifact. Its first line after the marker: `Self-approved under /execute.`
- **issue-to-task — `Undocumented decision:` lines.** Before drafting the entry, grep `arch/` on `origin/<default>` for one already recording it. If present, the plan says `already recorded as ADR-<n>` and drafts nothing.
- **ADR numbering — reserve, don't count.** Whenever a stage gives an `arch/` entry a number (a plan drafting one, ship authoring one, merge renumbering one), it reserves the number atomically so parallel issues never pick the same one:

  ```bash
  git fetch origin <default>
  LOCKS="$(git -C <worktree> rev-parse --path-format=absolute --git-common-dir)/execute-locks"; mkdir -p "$LOCKS/adr"
  top=$(git ls-tree -r --name-only origin/<default> arch/ | sed -nE 's#.*/0*([0-9]+)-.*#\1#p' | sort -n | tail -1)
  res=$(ls "$LOCKS/adr" 2>/dev/null | sort -n | tail -1)
  n=$(( ${top:-0} > ${res:-0} ? ${top:-0} : ${res:-0} )); n=$((n+1))
  until mkdir "$LOCKS/adr/$n" 2>/dev/null; do [ -d "$LOCKS/adr/$n" ] || exit 1; n=$((n+1)); done
  echo "issue-<N>" > "$LOCKS/adr/$n/owner"
  ```

  This replaces "one past the highest file" in `/issue-to-task` step 6 and `/ship` step 5 under `/execute`. A gap in the numbering (an issue that reserved and then failed or dropped its entry) is allowed; a number is never reused. A stage that already holds a reservation for its entry reuses it rather than taking a second.
- **implement under `/execute` — suite, not the check.** Commit all work at the stage boundary. Run the test suite; do not run the issue's verification check (verify runs it next on the same tree). An intermediate suite run here or in fix may be narrowed (`-m "not integration"`, a dummy port) when a port lock is held by another issue, but must say so.
- **implement precondition — no test command.** Take it from the issue's `## Verification` block. If that is empty too, return `blocked`. Never invent one.
- **implement step 3 — create the branch.** The worktree is already on the issue's linked branch, whose name the prompt gives. Confirm with `git -C <worktree> branch --show-current` and do **not** run `gh issue develop`: it would create and link a second branch.
- **implement step 9 — off-plan file or decision.** If the issue's goal requires it, post an amended plan comment — same marker, first line `Amended: <file or decision> — <why the goal requires it>` — and continue; the amendment is the new approved plan. If the goal does not require it, file a new issue for it and stay in scope.
- **verify step 2 — no check.** As for issue-to-task: write one into the issue.
- **verify step 4 — check fails.** Post the failure as described under Advance, then return `failed`.
- **verify step 5 — check cannot run here** (credentials, a browser, hardware, a service nobody can start). Return `blocked` with exactly what it needs.
- **verify under `/execute` — scope and evidence.**
  - Before running the check, `git status --porcelain` (untracked files included, ignored excluded) must print nothing; otherwise return `failed` listing the files. Evidence about a tree that is not committed is evidence about nothing.
  - Run the issue's full check, never narrowed (see Shared resources).
  - For every test the branch adds, show it failing on `origin/<default>` (red-on-base) and record the command and output. A test that passes on the base proves nothing; this is the step that catches a check that cannot fail.
  - Check ADR consequences for contradictions, as the skill says.
  - Do **not** re-diff the branch against the plan; ship step 4 owns plan conformance.
  - If the diff edits `README.md`, run each command it adds or changes that can run here (skip ones the check already exercised, and anything that needs credentials), and post the results with the evidence. A README command that fails is a failed check.
  - Record in the evidence comment: `HEAD` (`git rev-parse HEAD`), `tree: <git rev-parse HEAD^{tree}>`, and the line `full check: yes` (or what was narrowed and why).
- **ship steps 3–4 — reviews.** Under `/execute`, do **not** invoke `/code-review`: it forks a named agent whose findings have been lost or delivered after merge. After the step-2 commit, launch two **unnamed** Agent calls in one message and wait for both before step 5:
  - (a) a correctness review of `git diff origin/<default>...HEAD` in the worktree, read-only: hunt for bugs, regressions, hangs and resource leaks; verify each finding before reporting it; report every finding with file:line and a concrete failure scenario, uncapped, or say "no findings";
  - (b) the step-4 plan-conformance check.

  If either returns no usable result, re-run it once; if it fails again, return `failed`. Never proceed without both. List both reviews' findings, and say how they ran, in the PR body.
- **ship step 6 — evidence on an identical tree.** Re-run the check unless **all** of these hold: verify's evidence comment is newer than the last commit; `git rev-parse HEAD^{tree}` equals the `tree:` it recorded; `git status --porcelain` is empty; and it says `full check: yes`. When all hold, the PR body quotes verify's command and full output, links the comment, and states `Evidence from verify on identical tree <hash>`. Any step-5 commit, any mismatch, or a missing field forces a fresh run. The ship skill's rule — evidence generated on the exact tree being shipped — is what this satisfies.
- **ship step 5 — ADR authored during ship.** Authorized. Write it at `**Status:** Accepted`, number it per the rule above, and list it in the PR body under `Decisions recorded` with the words `authored at ship time, not in the approved plan`, and in the result's `adrs_outside_plan`. That flag is the user's audit trail for an entry nobody approved live.
- **ship step 5 — a review blocker the stage cannot fix.** Post it as a failure (see Advance) and return `failed`.
- **ship step 7 — confirm before the PR.** Self-confirmed. The PR body contains the line `Opened under /execute.` — the merge stage and resume merge nothing without it.
- **ship step 11 — do not merge.** Still holds for the ship stage. Merging is the merge stage's job.
- **finish step 2 — dirty tree.** The main checkout must be clean; if it is not, return `blocked` naming the files. Never stash, clean or reset.

### The fix stage

Runs in the issue's worktree, with the failure in its prompt. It is not a fresh implementation: keep the branch's tests and changes, reproduce the recorded failure, fix it at the root within the approved plan's file list (an off-plan need follows the implement step 9 gate), run the full suite, and stop. Never loosen an assertion, skip a test or suppress an error to get to green. The next stage — verify — is what judges the fix.

### The merge stage

Runs in the issue's worktree, one at a time across the whole run, and only for a PR whose body contains `Opened under /execute.`

1. `git fetch origin <default>` and `git merge origin/<default>`. On a conflict, resolve only within the plan's file list; a conflict outside it returns `failed`.
2. **Renumber on collision.** If an `arch/` file this branch adds has a number already on `origin/<default>`, rename it to a number taken through the reservation loop under "ADR numbering", fix its `# ADR-<n>` header and every reference in the diff, commit, update the PR body's `Decisions recorded` line, and post `Amended: ADR renumbered <old> → <new>; another entry landed first` as a plan comment. Never touch an entry already on the default branch.
3. If step 1 or 2 changed anything, re-run the issue's verification check and the suite. On a failure, post it (see Advance) and return `failed` with the diagnosis saying the re-check failed. On a pass, confirm `git status --porcelain --untracked-files=no` is empty — every edit committed — then push. After a renumber, also confirm `git show origin/<branch>:<new path>` begins with `# ADR-<new>`.
4. `gh pr view <PR> --json mergeable,mergeStateStatus`. Retry for up to a minute while it reads `UNKNOWN`. Merge only on `MERGEABLE`: `gh pr merge <PR> --squash`. Do not pass `--delete-branch`; `/finish` deletes branches, after the worktree is gone.
5. `gh pr view <PR> --json state`. Return `ok` **only** when it reads `MERGED`. A merge queue or auto-merge that left it `OPEN` returns `blocked` with the `mergeStateStatus`.
6. If branch protection requires a human review, return `needs_user`. Never bypass protection, never `--admin`.

Merge (here) and finish (in the main checkout) may run at the same time, and share one ref store. In both, a git command that fails with `cannot lock ref` or an existing `.lock` file is retried up to three times, 5 seconds apart, before it counts as a failure; such a retry is not a strike.

## Shared resources

Worktrees have different directory names, so each gets its own compose project — but they all publish the same host ports. Before any step that starts a service on a fixed host port, take an exclusive lock, and hold it from `up` through `down`:

```bash
LOCKS="$(git -C <worktree> rev-parse --path-format=absolute --git-common-dir)/execute-locks"
mkdir -p "$LOCKS" && mkdir "$LOCKS/port-<port>" && echo "issue-<N> $(date -u +%FT%TZ)" > "$LOCKS/port-<port>/owner"
```

- `mkdir` failing because the directory exists means held: poll every 15 seconds for up to 10 minutes, then return `blocked` naming the holder. Any other failure fails the step.
- Run compose from the worktree with `-p issue-<N>`. Release only after `docker compose -p issue-<N> down -v` succeeds and after killing any server the stage backgrounded. Never return with a lock held; if you must, list it in `locks_held`.
- Before `up`, if `docker ps --filter publish=<port> -q` is non-empty and the lock is free, something outside this run holds the port: return `blocked` naming the container. Never stop a container this run did not start.
- **Any run that connects to a fixed host port takes the lock** — not only `compose up`, but the issue's check and any test run that reaches a service (for job-scrape: `pytest` without `-m "not integration"`). It runs only against this issue's own compose project, never against another issue's container.
- **Evidence runs are never narrowed.** Verify, ship step 6 and merge step 3 run the full check and suite: no `-m "not integration"`, no dummy port, unless the issue's check itself says so. If the lock is held, wait for it as above.
- The orchestrator removes any `port-*` lock whose `owner` names an issue whose stage has returned, after confirming that issue's compose project has no running containers. The sweep never touches `adr/`; the orchestrator removes `adr/<n>` only once ADR-`<n>` is on `origin/<default>`, or at the end of the run.

`uv sync` in several worktrees shares uv's cache safely; each worktree gets its own `.venv`.

## Truly breaking

The only reasons a stage returns `needs_user` or the orchestrator interrupts the user:

1. **A supersede.** The plan would contradict an ADR the issue is constrained by.
2. **An unresolvable fork.** The issue and its spec disagree, or admit two readings, whose results differ in schema, public interface, wire format or dependency, and neither `arch/` nor the issue settles it.
3. **A dependency closed as not planned.**
4. **An irreversible act beyond this branch.** Dropping or rewriting shared data, force-pushing a shared branch, deleting anything but this issue's merged branch, stopping a container or process this run did not start.
5. **A merge that needs a human.** Branch protection requiring a review.
6. **A duplicate ADR number on the default branch.**
7. **The runaway guard.** Issues filed by workers this run exceed `max(5, 2 × the candidates at start)`.

Everything else the stage decides, and records the decision in the plan comment, the PR body, or the result. "I would have liked to ask" is not a reason; "a wrong guess here produces a different schema" is.

## New issues

A stage files an issue when it finds necessary work outside its issue's scope: a bug noticed in passing, a spec gap, a missing prerequisite. Before filing, `gh issue list --state open --limit 1000 --search "<key terms>"` for a duplicate. Use the `/spec-to-issues` template — Goal, Files likely involved, Out of scope, Verification, `Spec:`, `Constrained by`, `Depends on` — the parent issue's labels, and `specs/<slug>.md §n` citations. Add `Found while working #<N> under /execute.` and list the number in `new_issues`.

If the parent cannot finish without it, add `Depends on #<new>` to the parent's body and return `waiting_dependency`. The orchestrator re-fetches, schedules the new issue, and resumes the parent at the same stage once the new issue closes. Otherwise the parent carries on and the new issue is worked in its own turn.

## Rules for every stage

- **Posting a failure** ("see Advance" above): a stage that returns `failed` first posts its diagnosis and its verbatim failing output to the issue under `<!-- unit-of-work:execute-failure -->`, and puts the comment's URL in `failure_comment`.
- **Sentinel.** Put this file's last line, `END STAGE-RULES v1`, at the end of your result's `summary`. A result without it counts as `failed`: it means the rules were not read.
- Every command runs in the directory the prompt names, with `cd <dir> &&` or `git -C <dir>`.
- No stage proceeds on an open dependency, on a `/code-review` handle, or past a failing check.
- Every self-answered gate leaves a trace on the issue or the PR. State that lives only in an agent's context does not exist.
- Never bypass branch protection, never `--admin`, never force-push, never rewrite an `arch/` entry already on the default branch.
- Root causes only. Never loosen an assertion, skip a test or suppress an error to get to green.
- Report faithfully. A `blocked` or `failed` result says so, with its diagnosis.


END STAGE-RULES v1

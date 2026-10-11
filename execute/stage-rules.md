# Stage rules for /execute

Every stage the `/execute` orchestrator launches reads this file in full before its first command. It binds the stage alongside the stage's own skill. The orchestrator reads it too, at the start of every run.

## Gates

Each one is a point where a chained skill asks the user. Under `/execute` the stage answers it as written here, and leaves the answer where a human can audit it.

- **issue-to-task step 2 — open dependency.** Never proceed. Return `blocked`, naming it. If this fires, the graph was wrong.
- **issue-to-task step 4 — restatement.** Self-confirm. If writing the restatement shows the issue is ambiguous, edit the issue with `gh issue edit` so it says what it means, re-read it, and name the edit in the plan comment. Fix the issue, not the conversation.
- **issue-to-task — no verification check.** Write one into the issue under `## Verification`, derived from the issue's goal and the spec: a command and its expected output.
- **issue-to-task steps 5–9 — plan mode and approval.** There is no plan mode. Explore read-only, write the plan as text, gate it on the six criteria — fixing any that fail — and post it with the `<!-- unit-of-work:approved-plan -->` marker. The comment is the approval artifact. Its first line after the marker: `Self-approved under /execute.`
- **issue-to-task steps 6 and 8 — plan shape.** Say what changes and why, file by file, in prose; implement writes the code. No code beyond a signature or a one-line expression, and never a diff. Approach and Test plan together stay within about 80 lines; ADR drafts do not count. Each six-criteria verdict is one line.
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
- **implement step 9 — off-plan file or decision.** If the issue's goal requires it, post an amendment and continue; a `README.md` edit the change makes necessary always counts as required, since issue-to-task step 6 puts it in the same diff, never a follow-up. If the goal does not require it, file a new issue for it and stay in scope.
- **Amendments, and what "the approved plan" means.** An amendment is a comment with the same marker whose first line after it is `Amended: <what> — <why the goal requires it>`. It states only its delta: the files it adds, removes or renames, and the full text of any ADR draft it adds or rewrites; a renumber names only the old and new path. Any stage that reads the approved plan takes the last marked comment that is not an amendment and applies every later amendment in order. A fix that stays inside the plan's files and decisions needs no amendment.
- **verify step 2 — no check.** As for issue-to-task: write one into the issue.
- **verify step 4 — check fails.** Post the failure as described under Advance, then return `failed`.
- **verify step 5 — check cannot run here** (credentials, a browser, hardware, a service nobody can start). Return `blocked` with exactly what it needs.
- **verify under `/execute` — scope and evidence.**
  - Before running the check, `git status --porcelain` (untracked files included, ignored excluded) must print nothing; otherwise return `failed` listing the files. Evidence about a tree that is not committed is evidence about nothing.
  - Run the issue's full check, never narrowed (see Shared resources).
  - **Red on base.** For every test the branch adds, show it failing on `origin/<default>`: overlay the branch's changed test files on a detached worktree of `origin/<default>` outside the repository, run only the new tests with the shortest traceback the runner offers, then remove that worktree. A test that passes on the base proves nothing; this is the step that catches a check that cannot fail. List each new test that passes on the base as `passes on base: <test>`, and return `failed` when the approved plan's test plan names or describes it without saying it passes there. One the plan does not cover, such as a fix-round regression test for a defect the branch itself introduced, is only listed.
  - Check ADR consequences for contradictions, as the skill says.
  - Do **not** re-diff the branch against the plan; ship step 4 owns plan conformance.
  - If the diff edits `README.md`, run each command it adds or changes that can run here (skip ones the check already exercised, and anything that needs credentials), and post the results with the evidence. A README command that fails is a failed check.
  - **The evidence comment holds only:** the title `## Verification evidence: issue #<N>`; `HEAD: <git rev-parse HEAD>`, `tree: <git rev-parse HEAD^{tree}>` and `full check: yes` (or what was narrowed and why); each check command with its complete output, verbatim; the red-on-base command with each new test's failure line, verbatim, and any `passes on base:` lines; the README command results, when run; and each ADR consequence the check's output contradicts, or `No ADR consequence contradicted.` Nothing else: no tree-state, lock, compose or teardown logs, no restated check, no narration.
- **ship steps 3–4 — reviews.** Under `/execute`, do **not** invoke `/code-review`: it forks a named agent whose findings have been lost or delivered after merge. After the step-2 commit, launch two **unnamed** Agent calls in one message — (a) at effort `high`, (b) at `medium`, both at `max` when the issue is labeled `max-effort` — and wait for both before step 5:
  - (a) a correctness review of `git diff origin/<default>...HEAD` in the worktree, read-only: hunt for bugs, regressions, hangs and resource leaks; confirm each finding before reporting it — trace the code path, or reproduce it with a scratch test outside the worktree, never a benchmark; report every finding with file:line and a concrete failure scenario, uncapped, or say "no findings";
  - (b) the step-4 plan-conformance check.

  If either returns no usable result, re-run it once; if it fails again, return `failed`. Never proceed without both.
- **ship step 5 — one fix round.** Fix every finding that matters in one pass, with a test for each one a test can show, written first and seen failing before the fix, and commit. If those commits change behaviour, not only tests, comments or docs, launch review (a) again — same instructions and effort, unnamed — over just `git diff <the reviewed HEAD>..HEAD`, re-running it once if it returns nothing usable; fix what it finds in place or treat it as a blocker (below), and never review a third time. A fix inside the plan's files and decisions needs no amendment — list it in the PR body — and neither does an ADR authored here, which the gate below covers. Any other off-plan file or decision, a `README.md` edit included, follows the implement step 9 gate.
- **ship step 6 — evidence on an identical tree.** Re-run the check unless **all** of these hold: verify's evidence comment is newer than the last commit; `git rev-parse HEAD^{tree}` equals the `tree:` it recorded; `git status --porcelain` is empty; and it says `full check: yes`. When all hold, the PR body links that comment, quotes each check command with its final result line, and states `Evidence from verify on identical tree <hash>`. Any step-5 commit, any mismatch, or a missing field forces a fresh run. The ship skill's rule — evidence generated on the exact tree being shipped — is what this satisfies.
- **ship step 5 — ADR authored during ship.** Authorized. Write it at `**Status:** Accepted`, number it per the rule above, and list it in the PR body under `Decisions recorded` with the words `authored at ship time, not in the approved plan`, and in the result's `adrs_outside_plan`. That flag is the user's audit trail for an entry nobody approved live.
- **ship steps 5–6 — a review blocker the stage cannot fix, or a check that fails after the fix round.** Post it as a failure (see Advance) and return `failed`.
- **ship step 7 — confirm before the PR.** Self-confirmed. The PR body contains the line `Opened under /execute.` — the merge stage and resume merge nothing without it.
- **ship step 9 — the PR body.** Only these, in order: `Closes #<N>`; `Opened under /execute.`; what changed and why in at most ten lines — the diff shows how, so never walk through it; the evidence per step 6 — on a fresh run, `HEAD`, `tree` and each check command with its complete output, verbatim, and nothing else from the run; each review finding in one line, marked `fixed in <sha>` or with why it was left, and one line on how the reviews ran; departures from the plan, one line each; and the `README updated:` and `Decisions recorded:` lines when they apply, keeping step 5's ship-time flag.
- **ship step 12 — do not merge.** Still holds for the ship stage. Merging is the merge stage's job.
- **finish step 2 — dirty tree.** The main checkout must be clean; if it is not, return `blocked` naming the files. Never stash, clean or reset.
- **finish step 8 — evidence for a quick-lane issue.** It lives in the PR body, not in an issue comment: link it there instead of reporting none.

### The quick stage

Runs in the issue's worktree for a fast-lane issue — labeled `small` but not `max-effort` — in place of plan, implement, verify and ship. The ADR numbering gate, Shared resources and the rules for every stage apply. For such an issue, wherever a later stage reads "the plan's file list" or "the approved plan's file list", it means the files the issue names or implies plus those its PR changes.

1. Read the issue, the code it names, and only the spec sections and ADRs it cites. If the change needs a file the issue does not name or imply, a decision the issue does not state, or more than one focused change, post the reason to the issue as `Escalated from the fast lane: <reason>` and return `escalate` with it in `diagnosis` — before editing anything, or on a resume, before editing further.
2. Write the test the issue's check implies, run it, and keep its failing line; when the worktree already holds this issue's work (a resume or a relaunch), keep that work and get the line by verify's red-on-base procedure instead. Implement, run the suite and the issue's full check, and write any ADR the issue's stated decision earns — at `**Status:** Accepted`, from the reason the issue gives — and any `README.md` edit the change makes necessary.
3. Commit all work. Launch one unnamed review at effort `medium` over `git diff origin/<default>...HEAD`: review (a)'s terms, plus conformance against the issue — every changed file is one it names or implies, its check is satisfied, and its decision is recorded. Fix what matters in one pass, with no second review, and re-run the check if anything changed. A finding that matters and cannot be fixed in that pass is posted as a failure, and the stage returns `failed` without opening the PR.
4. Push and open the PR, writing its body by the PR-body gate with these substitutions: the evidence is `HEAD`, `tree`, the test's failing line and the check's command with its complete output; departures are from the issue rather than a plan; and an ADR written here is flagged `authored from the issue, not a plan` and listed in `adrs_outside_plan`.

Stop once the PR is open. A check that cannot be made to pass is `failed`, posted as a failure.

### The fix stage

Runs in the issue's worktree, with the failure in its prompt. It is not a fresh implementation: keep the branch's tests and changes, reproduce the recorded failure, fix it at the root within the approved plan's file list (an off-plan need follows the implement step 9 gate), run the full suite, commit all work, and stop: verify fails a dirty tree. Never loosen an assertion, skip a test or suppress an error to get to green. The next stage — verify — is what judges the fix.

### The merge stage

Runs in the issue's worktree, one at a time across the whole run, and only for a PR whose body contains `Opened under /execute.`

1. `git fetch origin <default>` and `git merge origin/<default>`. On a conflict, resolve only within the plan's file list; a conflict outside it returns `failed`.
2. **Renumber on collision.** If an `arch/` file this branch adds has a number already on `origin/<default>`, rename it to a number taken through the reservation loop under "ADR numbering", fix its `# ADR-<n>` header and every reference in the diff, commit, update the PR body's `Decisions recorded` line, and post an amendment: `Amended: ADR renumbered <old> → <new> — another entry landed first`, naming the old and new path. Never touch an entry already on the default branch.
3. If step 1 or 2 changed anything, re-run the issue's verification check and the suite; on a failure, post it (see Advance) and return `failed` with the diagnosis saying the re-check failed. Then, changed or not — a fix stage leaves its commits only in this worktree — confirm `git status --porcelain --untracked-files=no` is empty, `git push origin HEAD`, and confirm `git rev-parse HEAD` equals `git rev-parse origin/<branch>`, so the squash merges what was checked. After a renumber, also confirm `git show origin/<branch>:<new path>` begins with `# ADR-<new>`.
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

A stage files an issue when it finds necessary work outside its issue's scope: a bug noticed in passing, a spec gap, a missing prerequisite. Before filing, `gh issue list --state open --limit 1000 --search "<key terms>"` for a duplicate. Use the `/spec-to-issues` template — Goal, Files likely involved, Out of scope, Verification, `Spec:`, `Constrained by`, `Depends on` — the parent issue's labels except `small` (which a new issue gets only when it meets `/brainstorm`'s small test), and `specs/<slug>.md §n` citations. Add `Found while working #<N> under /execute.` and list the number in `new_issues`.

If the parent cannot finish without it, add `Depends on #<new>` to the parent's body and return `waiting_dependency`. The orchestrator re-fetches, schedules the new issue, and resumes the parent at the same stage once the new issue closes. Otherwise the parent carries on and the new issue is worked in its own turn.

## Rules for every stage

- **Posting a failure** ("see Advance" above): a stage that returns `failed` first posts its diagnosis and its verbatim failing output to the issue under `<!-- unit-of-work:execute-failure -->`, and puts the comment's URL in `failure_comment`.
- **Sentinel.** Put this file's last line, `END STAGE-RULES v1`, at the end of your result's `summary`. A result without it counts as `failed`: it means the rules were not read.
- Every command runs in the directory the prompt names, with `cd <dir> &&` or `git -C <dir>`.
- No stage proceeds on an open dependency, on a `/code-review` handle, or past a failing check.
- Every self-answered gate leaves a trace on the issue or the PR. State that lives only in an agent's context does not exist.
- Never bypass branch protection, never `--admin`, never force-push, never rewrite an `arch/` entry already on the default branch.
- Root causes only. Never loosen an assertion, skip a test or suppress an error to get to green.
- No benchmarks, load scripts or one-off measurements unless the issue's goal or check asks for them; a test is how a claim is proven.
- Report faithfully. A `blocked` or `failed` result says so, with its diagnosis.


END STAGE-RULES v1

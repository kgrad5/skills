---
name: execute
description: Drives every open GitHub issue, or only the issues named, through the whole unit-of-work chain autonomously — /issue-to-task, /implement, /verify, /ship, merge, /finish — running each stage in its own fresh subagent at the effort that stage deserves, in parallel across every issue whose dependencies are closed, filing new issues when the work reveals them and working those too, and stopping to ask only on a truly breaking question. This skill should be used when the user says "execute", "run the backlog", "work all the issues", "execute issues 3 4 5", or "keep going until the issues are done".
argument-hint: [issue-number ...] [--max-parallel <n>]
---

# execute

Guarantees that an unattended run of the backlog goes through every gate the manual loop has — plan, scope guard, evidence, review, recorded decisions — with a subagent answering the gates a human would have answered, and with every answer left on the issue or the pull request where a human can audit it later.

Invoking `/execute` is the user's **standing authorization** for four things the other skills otherwise refuse without a live yes: self-confirmed restatements and plans; ADR entries authored at ship time and marked `Accepted`, provided each is flagged as such in the PR body; pull requests opened without a confirmation prompt; and **squash-merging the pull requests this run opened** once review and verification pass. Nothing else is authorized. Every rule in the chained skills not answered under "Gates" below still binds.

## Preconditions

Check and report; never assume. Any failure stops the run before a single agent launches.

- `gh auth status` succeeds; `gh repo view --json nameWithOwner,defaultBranchRef` resolves. Use that default branch everywhere; never hardcode `main`.
- The main checkout is clean (`git status --porcelain` empty) and on the default branch. Record its absolute path. **No stage runs a command in the main checkout except the finish stage.**
- `git fetch origin --prune` succeeds.
- **Permission mode.** The skill cannot reliably read it, so state it: workers need a mode that does not prompt per tool (auto mode or bypass permissions). In default mode every worker's prompts queue in this session and the run crawls; say so and continue only if the user already said to. In `dontAsk` mode unapproved commands are silently denied and workers fail for no visible reason; **stop**.
- **Agent teams.** If `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` is set to anything but `0` (check the environment and `~/.claude/settings.json`), any *named* Agent call becomes an in-process teammate that runs in the main checkout, and nothing but the never-name rule below prevents that. Put it to the user with `AskUserQuestion`: disable it (set it to `"0"` in the settings `env` block and restart the session — recommended), or proceed relying on the never-name rule.

## Inputs

Arguments given: $ARGUMENTS

- Every argument matching `^#?[0-9]+$` is a **selected issue**. With none, the candidate set is every open issue, re-read throughout the run, until none remain.
- With some, the candidate set is exactly those, plus any issue a worker files during the run that a selected issue now depends on. Other new issues are filed and reported, not worked. A selected issue that depends on an **open, unselected** issue is reported blocked and never started.
- `--max-parallel <n>` caps concurrent stage agents. Default 100, which in practice launches every runnable stage at once. Merge and finish stages are always serial regardless.

## Effort per stage

Every stage is its own **fresh, unnamed Agent call** with the `effort` set from this table. The model is inherited. A fresh context per stage is the "one task per session" rule applied per stage; it works because every stage reads its inputs from the issue, the branch, the PR and the prompt the orchestrator writes — never from a previous agent's memory.

| Stage | Effort | Why |
|---|---|---|
| plan — `/issue-to-task` | `max` | Every later stage inherits its mistakes; it drafts the ADRs |
| implement — `/implement` | `high` | Executes an approved plan under a mechanical scope guard |
| verify — `/verify` | `max` | Evidence and ADR-contradiction checks are the definition of done |
| ship — `/ship` to an open PR | `max` | Review and plan conformance are the only scrutiny before an unattended merge |
| fix — repair a recorded failure, then hand back | `high` | Same footing as implement, with the failure in hand |
| merge — integrate base, re-check, squash-merge | `high` | Re-runs the check against a base that moved; renumbers ADRs on collision |
| finish — `/finish` | `low` | Mechanical gh and git steps whose refusals are already built in |

The orchestrator itself runs at the session's effort. New issues are filed inside whichever stage discovers the work, at that stage's effort.

## Procedure

### 1. Build the graph

```bash
gh issue list --state open --limit 1000 --json number,title,body,labels
```

Never omit `--limit`; the default is 30 and silently drops the oldest issues, which are the dependency roots.

**Dependencies.** For every line of a body matching `(?i)^\s*(depends on|blocked by)\b`, take **every** `#([0-9]+)` on that line — `Depends on #3, #4, #5, #6` is four edges, not one. Union across lines. For each dependency, `gh issue view <d> --json state,stateReason`:
- `CLOSED` with `COMPLETED` → satisfied.
- `CLOSED` with `NOT_PLANNED` → **truly breaking**: the issue was written against work that will never exist.
- `OPEN` → unsatisfied. Not found → blocked; report it.

**Print the edge list and the first wave before launching anything.** A wrong graph is cheap to see here and expensive to see in a PR.

### 2. Recover what an earlier run left behind

GitHub is the store, so a run must reconstruct each issue's stage from it alone. **Find an issue's PR this way and no other** — GitHub search is not a phrase match, and `Closes #1` also matches a PR that closes #12 and mentions "1 passed":

```bash
gh pr list --state all --limit 1000 --search "#<N> in:body" --json number,state,headRefName,body,mergedAt \
  | jq --arg n "<N>" '[.[] | select(.body | test("(?i)\\bcloses #" + $n + "\\b"))]'
```

and, when the issue has a linked branch (`gh issue develop --list <N>`), keep only PRs whose `headRefName` is that branch. More than one survivor → `blocked`, listing them.

Then take the **first** row that matches, for every candidate and every issue with a surviving branch or worktree:

| PR | Other state | Start at |
|---|---|---|
| `MERGED` | branch or worktree survives | finish (remove the worktree first) |
| `MERGED` | nothing survives | done |
| `OPEN`, body contains `Opened under /execute.` | — | ensure the worktree (from `headRefName`), then verify, then merge |
| `OPEN`, without that line | — | `blocked`: a PR this skill did not open is never merged by it |
| none | `<!-- unit-of-work:execute-failed -->` newer than the last plan comment | `failed`; reported, not restarted |
| none | no approved-plan comment | plan |
| none | approved plan, worktree or branch with changes beyond the base (committed or not) | verify — a failure routes to fix, which finishes the work |
| none | approved plan, nothing implemented | implement |

"Ensure the worktree" means: reuse the one `git worktree list --porcelain` shows for the branch; else `git worktree add <path> <branch>`. Never remove a worktree on resume — it may be the only copy of uncommitted work — and never create a second one for the same branch.

Never skip verify on the strength of an old evidence comment. The comment describes a tree that may no longer exist.

### 3. Worktrees

One per issue, created by the orchestrator before the issue's first stage, never inside the repository:

```bash
git fetch origin <default>
gh issue develop <N> --base <default> --name <N>-<kebab-slug-of-title> \
  --checkout --worktree "<absolute parent of the main checkout>/<repo-name>.worktrees/issue-<N>"
```

`gh` does not resolve the path; pass it absolute. The branch is linked to the issue, starts with the issue number so every chained skill infers it, and lives in a sibling directory, so the main checkout never shows it as untracked. Do **not** use the Agent tool's `isolation: "worktree"`: it creates a new worktree per call, and a stage chain needs one worktree that outlives each stage.

### 4. Schedule

Keep a state per issue: `queued:<stage>`, `running:<stage>`, `waiting:merge`, `needs_user`, `waiting:dependency`, `blocked`, `failed`, `done`.

- **Runnable** means `queued` and every dependency satisfied.
- Launch the next stage of every runnable issue, up to `--max-parallel` stage agents at once.
- When a stage agent returns, read its result (format below), advance the issue (step 6), and immediately launch whatever became runnable. Do not wait for a wave to finish.
- **Merge and finish are serial.** At most one issue is in merge or finish at any moment. This keeps two ADRs from landing the same number and two merges from racing the same base.
- **Re-fetch the issue list** (step 1) after every finish **and** after any stage whose result lists `new_issues`. New issues enter as `queued:plan`. An issue in `waiting:dependency` returns to `queued` at its interrupted stage the moment its new dependency closes.
- `failed`, `blocked` and `needs_user` are never relaunched by recomputation. `needs_user` returns to `queued` when the user answers; `failed` and `blocked` only on a new `/execute`.

The run ends when no candidate is `queued`, `running` or `waiting`. If any are `failed`, `blocked` or `needs_user` at that point, **do not just stop**: put them to the user in one `AskUserQuestion` — retry with guidance, skip, or stop — then continue or report.

### 5. The stage prompt

Launch each stage with the Agent tool: `effort` from the table, **no `name`**, no `isolation`, in the background. With fork mode on (the interactive default) background is automatic and the tool has no `run_in_background` parameter; where the parameter exists, pass `true`. The orchestrator is notified when each one returns.

Every prompt carries:

- The issue number, the stage, the default branch, the **issue branch name**, the **PR number** when one exists, and any answer the user gave for this issue.
- **The working directory.** For every stage except finish: the absolute worktree path, and the rule *the shell's directory resets between calls; begin every command with `cd <worktree> &&` or use `git -C <worktree>`; never run a command against the main checkout*. For finish: the absolute main-checkout path and the same rule against it; the worktree no longer exists.
- **For fix, and for any stage re-run after a failure:** the previous failing stage's `diagnosis` and its verbatim failing output, copied into the prompt, plus the URL of the `<!-- unit-of-work:execute-failure -->` comment that holds them.
- **Skill rule.** Run the stage's skill with the Skill tool (`issue-to-task`, `implement`, `verify`, `ship`, `finish`), passing the issue number — and for finish, the PR number, which `/finish` accepts in place of searching. If the Skill tool is unavailable, read `~/.claude/skills/<name>/SKILL.md` and follow it exactly. The merge and fix stages have no skill; their instructions are below.
- **Tool rule.** Subagents have no `AskUserQuestion`, `EnterPlanMode` or `ExitPlanMode`. Where a skill uses one, apply the matching gate; where none matches and the question is truly breaking, return `needs_user`; otherwise decide, and record the decision where the skill would have recorded the user's answer.
- The **Gates**, **Truly breaking**, **New issues**, **Shared resources** and **Rules for every stage** sections below, verbatim.
- **Stop at the stage boundary.** Plan stops after posting the plan; implement after the suite passes; verify after posting evidence; ship after the PR is open; fix after its repair is committed or left in the tree; merge after the PR reads `MERGED`; finish after its report.
- **Result.** End with a single fenced `json` block and nothing after it:

  ```json
  {"issue": 7, "stage": "verify", "outcome": "ok|failed|blocked|waiting_dependency|needs_user",
   "summary": "one sentence", "diagnosis": "on any non-ok outcome: what failed and why",
   "failure_comment": "URL of the execute-failure comment, on failed",
   "question": "on needs_user: the question, the options, and the consequence of each",
   "pr": 12, "new_issues": [13], "adrs": ["arch/0013-....md"],
   "adrs_outside_plan": ["arch/...md"], "locks_held": []}
  ```

  A result that cannot be parsed counts as `failed` with the raw text as the diagnosis.

### 6. Advance

| Stage returned | Next |
|---|---|
| plan `ok` | implement |
| implement `ok` | verify |
| verify `ok`, no PR yet | ship |
| verify `ok`, PR open | merge |
| verify `failed` | fix, then verify |
| ship `ok` | `waiting:merge`; merge when the slot is free |
| ship `failed` on a review blocker | fix, then verify, then ship |
| merge `ok` | remove the worktree, then finish |
| merge `failed` because its re-check failed | fix, then verify, then merge — the PR already exists; never route back through ship |
| finish `ok` | `done`; re-fetch the issue list |
| plan, implement, fix, finish `failed`, or ship or merge `failed` for any other reason | relaunch the same stage once with the diagnosis; a second failure is strike two |
| any `waiting_dependency` | `waiting:dependency` (see New issues) |
| any `blocked` | `blocked`, with the diagnosis |
| any `needs_user` | ask the user now with `AskUserQuestion`, quoting the question and the issue; on an answer, relaunch the same stage with the answer in the prompt |

**Failures are recorded where the next stage can read them.** A stage that returns `failed` first posts its diagnosis and its verbatim failing output to the issue under `<!-- unit-of-work:execute-failure -->`, and puts the comment's URL in `failure_comment`. This is the one place `/verify` posts on a failure: under `/execute` the next fresh agent cannot see a failure any other way.

**Two strikes.** The second failure of the same stage for the same issue — counting a verify failure in merge's re-check as a verify failure — marks the issue `failed`. That stage posts `<!-- unit-of-work:execute-failed -->` linking both failure comments, so the next session — human or `/execute` — sees why without this conversation. Never a third attempt in the same run.

**Removing a worktree,** only after the PR reads `MERGED` and before finish, so `/finish` can delete the branch:

```bash
test -z "$(git -C <worktree> status --porcelain --untracked-files=no)" || { echo "tracked changes left in worktree"; exit 1; }
cd <worktree> && docker compose -p issue-<N> down -v --remove-orphans
git -C <main checkout> worktree remove --force <worktree> && git -C <main checkout> worktree prune
```

**The first line is a hard gate.** A modified or staged tracked file means a stage claimed work it never committed, so the merged PR is missing it. Never remove the worktree then. Diff the change against `origin/<default>`, mark the issue `blocked`, and report it; the fix goes through a new issue. Skip the compose line when the worktree has no compose file. If `down` fails, stop: leave the worktree and its locks, mark the issue `blocked`, and report the containers still running. `--force` on removal is for untracked `.venv/` and caches; the work itself is merged. The `-v` names the worktree's own project, never the main checkout's.

**After every merge,** check the default branch for duplicate ADR numbers:

```bash
git fetch origin <default>
git ls-tree -r --name-only origin/<default> arch/ | sed -nE 's#.*/([0-9]{4})-.*#\1#p' | sort | uniq -d
git ls-tree -r --name-only origin/<default> arch/ | grep '\.md$' | while read -r f; do
  n=$(basename "$f" | sed -nE 's/^0*([0-9]+)-.*/\1/p')
  h=$(git show "origin/<default>:$f" | sed -nE '1s/^# ADR-([0-9]+).*/\1/p')
  [ "$n" = "$h" ] || echo "heading mismatch: $f is headed ADR-$h"
done
```

The second check catches a renumber that moved the file but not its heading.

Any output is **truly breaking**: stop launching, report it, ask. `arch/` is append-only; a landed duplicate cannot be renumbered by a worker.

## Gates

Each one is a point where a chained skill asks the user. Under `/execute` the stage answers it as written here, and leaves the answer where a human can audit it.

- **issue-to-task step 2 — open dependency.** Never proceed. Return `blocked`, naming it. If this fires, the graph was wrong.
- **issue-to-task step 4 — restatement.** Self-confirm. If writing the restatement shows the issue is ambiguous, edit the issue with `gh issue edit` so it says what it means, re-read it, and name the edit in the plan comment. Fix the issue, not the conversation.
- **issue-to-task — no verification check.** Write one into the issue under `## Verification`, derived from the issue's goal and the spec: a command and its expected output.
- **issue-to-task steps 5–9 — plan mode and approval.** There is no plan mode. Explore read-only, write the plan as text, gate it on the six criteria — fixing any that fail — and post it with the `<!-- unit-of-work:approved-plan -->` marker. The comment is the approval artifact. Its first line after the marker: `Self-approved under /execute.`
- **issue-to-task — `Undocumented decision:` lines.** Before drafting the entry, grep `arch/` on `origin/<default>` for one already recording it. If present, the plan says `already recorded as ADR-<n>` and drafts nothing.
- **ADR numbering.** Always count against `origin/<default>` after `git fetch`, never the local default branch, which no worktree advances.
- **implement precondition — no test command.** Take it from the issue's `## Verification` block. If that is empty too, return `blocked`. Never invent one.
- **implement step 3 — create the branch.** The worktree is already on the issue's linked branch, whose name the prompt gives. Confirm with `git -C <worktree> branch --show-current` and do **not** run `gh issue develop`: it would create and link a second branch.
- **implement step 9 — off-plan file or decision.** If the issue's goal requires it, post an amended plan comment — same marker, first line `Amended: <file or decision> — <why the goal requires it>` — and continue; the amendment is the new approved plan. If the goal does not require it, file a new issue for it and stay in scope.
- **verify step 2 — no check.** As for issue-to-task: write one into the issue.
- **verify step 4 — check fails.** Post the failure as described under Advance, then return `failed`.
- **verify step 5 — check cannot run here** (credentials, a browser, hardware, a service nobody can start). Return `blocked` with exactly what it needs.
- **ship step 3 — `/code-review` returns a handle.** Wait for its findings. If the stage cannot obtain them, run the correctness review as an Agent call over `git diff origin/<default>...HEAD` and wait for its result; if that is impossible too, review the diff cold yourself and say so in the PR body. Never proceed on a handle.
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
2. **Renumber on collision.** If an `arch/` file this branch adds has a number already on `origin/<default>`, rename it to one past the highest across `arch/` and `arch/archive/` on `origin/<default>`, fix its `# ADR-<n>` header and every reference in the diff, commit, update the PR body's `Decisions recorded` line, and post `Amended: ADR renumbered <old> → <new>; another entry landed first` as a plan comment. Never touch an entry already on the default branch.
3. If step 1 or 2 changed anything, re-run the issue's verification check and the suite. On a failure, post it (see Advance) and return `failed` with the diagnosis saying the re-check failed. On a pass, confirm `git status --porcelain --untracked-files=no` is empty — every edit committed — then push. After a renumber, also confirm `git show origin/<branch>:<new path>` begins with `# ADR-<new>`.
4. `gh pr view <PR> --json mergeable,mergeStateStatus`. Retry for up to a minute while it reads `UNKNOWN`. Merge only on `MERGEABLE`: `gh pr merge <PR> --squash`. Do not pass `--delete-branch`; `/finish` deletes branches, after the worktree is gone.
5. `gh pr view <PR> --json state`. Return `ok` **only** when it reads `MERGED`. A merge queue or auto-merge that left it `OPEN` returns `blocked` with the `mergeStateStatus`.
6. If branch protection requires a human review, return `needs_user`. Never bypass protection, never `--admin`.

## Shared resources

Worktrees have different directory names, so each gets its own compose project — but they all publish the same host ports. Before any step that starts a service on a fixed host port, take an exclusive lock, and hold it from `up` through `down`:

```bash
LOCKS="$(git -C <worktree> rev-parse --path-format=absolute --git-common-dir)/execute-locks"
mkdir -p "$LOCKS" && mkdir "$LOCKS/port-<port>" && echo "issue-<N> $(date -u +%FT%TZ)" > "$LOCKS/port-<port>/owner"
```

- `mkdir` failing because the directory exists means held: poll every 15 seconds for up to 10 minutes, then return `blocked` naming the holder. Any other failure fails the step.
- Run compose from the worktree with `-p issue-<N>`. Release only after `docker compose -p issue-<N> down -v` succeeds and after killing any server the stage backgrounded. Never return with a lock held; if you must, list it in `locks_held`.
- Before `up`, if `docker ps --filter publish=<port> -q` is non-empty and the lock is free, something outside this run holds the port: return `blocked` naming the container. Never stop a container this run did not start.
- The orchestrator removes any lock whose `owner` names an issue whose stage has returned, after confirming that issue's compose project has no running containers.

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

- Every command runs in the directory the prompt names, with `cd <dir> &&` or `git -C <dir>`.
- No stage proceeds on an open dependency, on a `/code-review` handle, or past a failing check.
- Every self-answered gate leaves a trace on the issue or the PR. State that lives only in an agent's context does not exist.
- Never bypass branch protection, never `--admin`, never force-push, never rewrite an `arch/` entry already on the default branch.
- Root causes only. Never loosen an assertion, skip a test or suppress an error to get to green.
- Report faithfully. A `blocked` or `failed` result says so, with its diagnosis.

## Hard rules for the orchestrator

- **Never name a stage agent**, and never pass `isolation`. Named Agent calls become teammates running in the main checkout when agent teams are on.
- **One stage per agent, at that stage's effort.** Never collapse the chain into one long agent; the effort table and the fresh context are the point.
- **Merge and finish are serial.**
- **Merge only PRs this run's ship stage opened**, identified by `Opened under /execute.` in the body.
- **Find a PR only by the filtered lookup in step 2.**
- **Two strikes and the issue is `failed`,** with the diagnosis on the issue. No third attempt in the same run.
- **Report faithfully.** A `blocked` or `failed` issue is reported as such, never folded into a success count.

## Output

As each issue finishes, one line: issue, outcome, PR URL. At the end:
- Issues done, each with its PR URL and merge commit.
- Issues failed, blocked, or waiting on the user, each with its diagnosis or question.
- Issues filed by workers, by number, and whether they were worked.
- `arch/` files recorded, by path, flagging every one in `adrs_outside_plan`.
- Issue bodies edited to remove ambiguity, by number.
- Locks waited on or swept, and worktrees removed.

## Next step

Read the merged diffs in the GitHub UI, starting with every PR that carries an ADR authored outside its plan. For failed or blocked issues, fix the issue body or answer the question, then run `/execute <numbers>` again; it resumes from GitHub.

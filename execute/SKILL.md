---
name: execute
description: Drives every open GitHub issue, or only the issues named, through the whole unit-of-work chain autonomously — /issue-to-task, /implement, /verify, /ship, merge, /finish — running each stage in its own fresh subagent at the effort that stage deserves, in parallel across every issue whose dependencies are closed, filing new issues when the work reveals them and working those too, and stopping to ask only on a truly breaking question. This skill should be used when the user says "execute", "run the backlog", "work all the issues", "execute issues 3 4 5", or "keep going until the issues are done".
argument-hint: [issue-number ...] [--max-parallel <n>]
---

# execute

Guarantees that an unattended run of the backlog goes through every gate the manual loop has — plan, scope guard, evidence, review, recorded decisions (in the fast lane, the issue's own scope stands in for the plan) — with a subagent answering the gates a human would have answered, and with every answer left on the issue or the pull request where a human can audit it later.

Invoking `/execute` is the user's **standing authorization** for four things the other skills otherwise refuse without a live yes: self-confirmed restatements and plans; ADR entries authored at ship time, or by the quick stage from the issue's stated decision, and marked `Accepted`, provided each is flagged as such in the PR body; pull requests opened without a confirmation prompt; and **squash-merging the pull requests this run opened** once review and verification pass. Nothing else is authorized. Every rule in the chained skills not answered under "Gates" in `stage-rules.md` still binds.

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
- `--max-parallel <n>` caps concurrent stage agents. Default 100, which in practice launches every runnable stage at once. Merges are always serial among themselves, and so are finishes, regardless.

## Effort per stage

Every stage is its own **fresh, unnamed Agent call** with the `effort` set from this table, or `max` under max mode below. The model is inherited. A fresh context per stage is the "one task per session" rule applied per stage; it works because every stage reads its inputs from the issue, the branch, the PR and the prompt the orchestrator writes — never from a previous agent's memory.

| Stage | Effort | Why |
|---|---|---|
| quick — the fast lane, for a fast-lane issue | `medium` | Plans, builds, checks and opens the PR in one pass for a change its issue already pins down; its one review runs at `medium` |
| plan — `/issue-to-task` | `high` | Every later stage inherits its mistakes; the plan is prose of about 80 lines, and `high` is ample for that |
| implement — `/implement` | `high` | Writes the code of an issue too big for the fast lane, under a mechanical scope guard |
| verify — `/verify` | `low` | Mechanical: runs the check and red-on-base and posts their output by the fixed steps in `stage-rules.md` |
| ship — `/ship` to an open PR | `medium` | Triages two review subagents — correctness at `high`, plan conformance at `medium` — fixes once and opens the PR |
| fix — repair a recorded failure, then hand back | `high` | A real failure, often a subtle one, in hand |
| merge — integrate base, re-check, squash-merge | `medium` | Re-runs the check against a base that moved; renumbers ADRs on collision |
| finish — `/finish` | `low` | Mechanical gh and git steps whose refusals are already built in |

The orchestrator itself runs at the session's effort; `medium` is enough, since it schedules and the stages do the thinking. New issues are filed inside whichever stage discovers the work, at that stage's effort.

**The fast lane.** A fast-lane issue — one labeled `small` but not `max-effort`; `/brainstorm`'s small path sets `small`, or add it by hand — runs one quick stage in place of plan, implement, verify and ship, then merge and finish as usual: minutes, not an hour. It suits a change whose issue already pins down the files, the behaviour and the check. A quick stage that finds the issue bigger than that returns `escalate` before editing anything, and the issue takes the full chain.

**Max mode.** An issue labeled `max-effort` — `/brainstorm --max` sets the label, or add it by hand when the work is critical — runs plan, implement, fix, ship with both its reviews, and merge at `max`, and never takes the fast lane. Verify and finish keep their efforts: they only run commands.

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
| none | `<!-- unit-of-work:execute-failed -->` newer than the last plan comment — or any, for a fast-lane issue | `failed`; reported, not restarted (delete that comment to retry) |
| none | labeled `small` but not `max-effort`, no approved-plan comment | quick, its prompt saying to keep any changes in the worktree |
| none | no approved-plan comment | plan |
| none | approved plan, worktree with uncommitted changes | implement, its prompt saying to keep those changes, finish the plan and commit |
| none | approved plan, branch with commits beyond the base | verify |
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
- **Merges are serial; finishes are serial; the two lanes overlap.** At most one issue is in merge at any moment, which keeps two merges from racing the same base. The next merge may start once the previous PR reads `MERGED`, the post-merge ADR checks have passed and its worktree is removed; it need not wait for that issue's finish. At most one finish runs at a time, since every finish works in the main checkout.
- **Re-fetch the issue list** (step 1) after every finish **and** after any stage whose result lists `new_issues`. New issues enter as `queued:plan`, or `queued:quick` when labeled `small` and not `max-effort`. An issue in `waiting:dependency` returns to `queued` at its interrupted stage the moment its new dependency closes.
- `failed`, `blocked` and `needs_user` are never relaunched by recomputation. `needs_user` returns to `queued` when the user answers; `failed` and `blocked` only on a new `/execute`.

The run ends when no candidate is `queued`, `running` or `waiting`. If any are `failed`, `blocked` or `needs_user` at that point, **do not just stop**: put them to the user in one `AskUserQuestion` — retry with guidance, skip, or stop — then continue or report.

### 5. The stage prompt

Launch each stage with the Agent tool: `effort` from the table, or `max` where max mode names the stage, **no `name`**, no `isolation`, in the background. With fork mode on (the interactive default) background is automatic and the tool has no `run_in_background` parameter; where the parameter exists, pass `true`. The orchestrator is notified when each one returns.

Every prompt carries:

- The issue number, the stage, the default branch, the **issue branch name**, the **PR number** when one exists, whether the issue is labeled `max-effort`, and any answer the user gave for this issue.
- **The working directory.** For every stage except finish: the absolute worktree path, and the rule *the shell's directory resets between calls; begin every command with `cd <worktree> &&` or use `git -C <worktree>`; never run a command against the main checkout*. For finish: the absolute main-checkout path and the same rule against it; the worktree no longer exists.
- **For fix, and for any stage re-run after a failure:** the previous failing stage's `diagnosis` and its verbatim failing output, copied into the prompt, plus the URL of the `<!-- unit-of-work:execute-failure -->` comment that holds them.
- **Skill rule.** Run the stage's skill with the Skill tool (`issue-to-task`, `implement`, `verify`, `ship`, `finish`), passing the issue number — and for finish, the PR number, which `/finish` accepts in place of searching. If the Skill tool is unavailable, read `~/.claude/skills/<name>/SKILL.md` and follow it exactly. The quick, merge and fix stages have no skill; their instructions are in `stage-rules.md`.
- **Tool rule.** Subagents have no `AskUserQuestion`, `EnterPlanMode` or `ExitPlanMode`. Where a skill uses one, apply the matching gate; where none matches and the question is truly breaking, return `needs_user`; otherwise decide, and record the decision where the skill would have recorded the user's answer.
- **The rules file, by reference.** The expanded absolute path of `~/.claude/skills/execute/stage-rules.md` and the line *first action: read this file in full; it binds this stage; end your result's `summary` with its last line*. Do **not** paste its sections into the prompt. A result whose summary lacks `END STAGE-RULES v1` counts as `failed`.
- **For merge:** the PR number, the plan's file list with amendments applied (for a quick-lane issue, the files the issue names or implies plus those its PR changes), and the issue's verification check, so the stage need not reconstruct them.
- **Stop at the stage boundary.** Quick stops once the PR is open; plan after posting the plan; implement after the suite passes; verify after posting evidence; ship after the PR is open; fix after its repair is committed; merge after the PR reads `MERGED`; finish after its report.
- **Result.** End with a single fenced `json` block and nothing after it:

  ```json
  {"issue": 7, "stage": "verify", "outcome": "ok|failed|blocked|waiting_dependency|needs_user|escalate",
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
| quick `ok` | `waiting:merge`; merge when the slot is free |
| quick `escalate` | remove the `small` label, then plan, with the escalation reason in its prompt |
| plan `ok` | implement |
| implement `ok` | verify |
| verify `ok`, no PR yet | ship |
| verify `ok`, PR open | merge |
| verify `failed` | fix, then verify |
| ship `ok` | `waiting:merge`; merge when the slot is free |
| ship `failed` on a review blocker or a check that fails after its fix round | fix, then verify, then ship |
| merge `ok` | remove the worktree, then finish |
| merge `failed` because its re-check failed | fix, then verify, then merge — the PR already exists; never route back through ship |
| finish `ok` | `done`; re-fetch the issue list |
| quick, plan, implement, fix, finish `failed`, or ship or merge `failed` for any other reason | relaunch the same stage once with the diagnosis; a second failure is strike two |
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

Then remove the ADR reservation for every entry now on `origin/<default>` (`$LOCKS/adr/<n>`; see the ADR numbering gate in `stage-rules.md`). Remove the rest at the end of the run. Never remove a reservation whose entry has not landed.

**Reaping finished agents,** after every finish and once more when the run ends: call `ListAgents` and `TaskStop` each `completed` subagent left from a returned stage. Never stop one shown as `running`, nor a `completed` one whose own child is still `running` — it is waiting on that child. Report the count in the run's output. Stages launch only unnamed agents, so no teammates should exist; if agent teams are on and some do, `TaskStop` each idle one by its name in the `members` list of `~/.claude/teams/<team>/config.json` — never `team-lead`, never one whose work is pending — and never edit that file or `inboxes/`: the session keeps teammates in memory, and an edited file loses the names `TaskStop` needs.

## Stage rules

The Gates, the quick, fix and merge stage instructions, Shared resources, Truly breaking, New issues and Rules for every stage live in `~/.claude/skills/execute/stage-rules.md`. **Read it in full at the start of every run**; it binds the orchestrator as well as every stage. Its last line is the sentinel `END STAGE-RULES v1`.

## Hard rules for the orchestrator

- **Never name a stage agent**, and never pass `isolation`. Named Agent calls become teammates running in the main checkout when agent teams are on.
- **One stage per agent, at that stage's effort.** Never collapse the chain into one long agent; the effort table and the fresh context are the point. The quick stage of a fast-lane issue is the one exception.
- **Merges are serial, and finishes are serial;** a merge may overlap a finish.
- **Merge only PRs this run's ship or quick stage opened**, identified by `Opened under /execute.` in the body.
- **Find a PR only by the filtered lookup in step 2.**
- **Two strikes and the issue is `failed`,** with the diagnosis on the issue. No third attempt in the same run.
- **Reap finished agents, never working ones,** after every finish and at the end of the run; a running subagent and its parent are never stopped.
- **Report faithfully.** A `blocked` or `failed` issue is reported as such, never folded into a success count.

## Output

As each issue finishes, one line: issue, outcome, PR URL, and each stage's wall time from its completion notice — the numbers to tune the effort table against. At the end:
- Issues done, each with its PR URL and merge commit.
- Issues failed, blocked, or waiting on the user, each with its diagnosis or question.
- Issues filed by workers, by number, and whether they were worked.
- `arch/` files recorded, by path, flagging every one in `adrs_outside_plan`.
- Issue bodies edited to remove ambiguity, by number.
- Locks waited on or swept, and worktrees removed.
- Finished agents reaped, and any teammates pruned.

## Next step

Read the merged diffs in the GitHub UI, starting with every PR that carries an ADR authored outside its plan. For failed or blocked issues, fix the issue body or answer the question, then run `/execute <numbers>` again; it resumes from GitHub.

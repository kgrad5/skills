# Unit-of-work skills

Seven skills that make [`unit-of-work-process.md`](unit-of-work-process.md) invocable instead of remembered: one task, from issue to merged, with a check that can pass or fail at every gate.

Installed at `~/.claude/skills/`, so they load in every session on this machine.

## The loop

```
/idea-to-spec  →  /spec-to-issues  →  ┐
                                      │ per issue, one session:
                                      └→ /issue-to-task 42
                                         [plan mode: explore, plan, approve]
                                         /implement
                                         /verify
                                         /ship            → PR opened, review pending
                                         /clear
                                            ⋮  [external code review]
                                         /merge 42        → squash, delete branch, issue closed
```

| Skill | Does | Refuses to |
|---|---|---|
| `/idea-to-spec` | Interviews you, writes `SPEC.md` | Overwrite an existing spec; spec a one-line change |
| `/spec-to-issues` | `SPEC.md` → ordered issues, each with a pass/fail check | Create anything before you approve the list; file an issue with no check |
| `/issue-to-task <n>` | Loads the issue, restates it, gates the plan on five criteria, posts the plan to the issue | Plan an issue whose restatement you say is wrong |
| `/implement` | Branches, writes the failing tests, implements | Run without an approved plan; touch a file the plan doesn't list |
| `/verify` | Runs the issue's check, shows command + full output | Summarize output; proceed on a failure |
| `/ship` | Reviews vs. the plan, commits, opens the PR | PR from the default branch; use evidence older than the last fix; merge |
| `/merge <n>` | Checks approval + CI, squash-merges, confirms the issue closed | Merge without `APPROVED` and green checks |

## Where the state lives

There is no state file. The **GitHub issue is the store**: `/issue-to-task` posts the approved plan to it as a comment marked `<!-- unit-of-work:approved-plan -->`, and `/implement` and `/ship` read it back.

This matters because the harness plan file cannot be relied on — it is named after the *session*, sits in a flat global directory, is swept after 30 days, and is frequently never written. The issue survives `/clear`, `/compact`, a new session, a new machine, and a different person. It is also why `/merge` works days later in a session that knows nothing about the work.

## What these skills deliberately do not do

Claude Code already does it:

| Step in the doc | Use instead |
|---|---|
| Start clean | `/clear`, `claude --permission-mode plan`, `Shift+Tab` |
| Explore | Plan mode's own exploration, the `Explore` subagent |
| Write the plan | Plan mode; `Ctrl+G` to edit it in your editor |
| Choose the leash | `ExitPlanMode`'s permission-mode prompt |
| Correctness review | `/code-review` — `/ship` delegates to it |
| Interrupt / rewind / compact | `Esc`, `Esc Esc` or `/rewind`, `/compact <instructions>` |
| Close out | `/clear` |

Four things from those sections are **not** keystrokes, so they live in the skills: the plan-approval checklist (`/issue-to-task`), plan-conformance review (`/ship` — `/code-review` never sees the issue or the plan), the two-strike rule (`/implement`), and the definition of done (`/merge`).

## Setup, once per repo

The skills check these and tell you what's missing, but they can't fix them:

1. `gh auth login` — five of the seven depend on it.
2. A check that runs by command. Without one, `/verify` has nothing to do.
3. `CLAUDE.md`, short — build/test/lint commands, style that differs from defaults, repo etiquette, gotchas. `/init` generates one; prune it hard.
4. `/permissions` — allowlist your test runner, linter, `git commit`, `git push`, and `gh`.

## Retrofitting an existing backlog

Issues not created by `/spec-to-issues` have no `## Verification` section. `/verify` proposes a check, confirms it with you, and writes it into the issue before running it — fixing the issue rather than the conversation, so the next session verifies the same thing. `SPEC.md` is optional everywhere except `/spec-to-issues`.

## Another machine

```bash
git clone git@github.com:kgrad5/skills ~/.claude/skills
```

## Editing

`SKILL.md` frontmatter is `name`, `description`, `argument-hint` only. `name` must equal its directory name.

No `allowed-tools`: it *restricts* the toolset rather than pre-approving it, and `/verify` and `/implement` must run project commands nobody can enumerate in advance. Permissions belong in `/permissions`.

The five-criteria checklist appears in both `/issue-to-task` and `/ship` — plan-vs-issue before the work, diff-vs-plan after. Change one, change the other.

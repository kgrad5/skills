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
                                         /ship            → PR opened
                                         /clear
                                            ⋮  [read the diff, merge in the GitHub UI]
                                         /finish 42       → branch deleted, issue closed
```

| Skill | Does | Refuses to |
|---|---|---|
| `/idea-to-spec` | Interviews you, writes `SPEC.md` with its ADR appendix | Overwrite an existing spec; drop or renumber an existing ADR; spec a one-line change |
| `/spec-to-issues` | `SPEC.md` → ordered issues, each with a pass/fail check and the ADRs that bind it | Create anything before you approve the list; file an issue with no check |
| `/issue-to-task <n>` | Loads the issue, restates it *and the ADRs binding it*, gates the plan on six criteria, posts the plan — and any ADR it drafts — to the issue | Plan an issue whose restatement you say is wrong; leave the plan silent about whether it decides architecture |
| `/implement` | Branches, writes the failing tests, implements, writes the plan's ADR into the log | Run without an approved plan; touch a file the plan doesn't list; make a decision the plan didn't draft |
| `/verify` | Runs the issue's check, shows command + full output | Summarize output; proceed on a failure |
| `/ship` | Reviews vs. the plan and the ADR log, commits, opens the PR | PR from the default branch; use evidence older than the last fix; open a PR whose diff decides architecture with nothing recorded; merge |
| `/finish [n]` | Confirms the merge happened, closes the issue if `Closes #N` didn't fire, confirms the ADRs landed on the default branch, deletes the branch | Touch anything if the PR isn't `MERGED`, or the tree is dirty; write the ADR log itself |

## Where the state lives

There is no state file. The **GitHub issue is the store**: `/issue-to-task` posts the approved plan to it as a comment marked `<!-- unit-of-work:approved-plan -->`, and `/implement` and `/ship` read it back.

The **decision log is the other store**, and the only one that outlives the issues: `## Appendix A — Architecture Decision Record` in `SPEC.md`, or `DECISIONS.md` where there is no spec. It is append-only. `/issue-to-task` reads it back to you at restatement time on every unit of work, which is what keeps it from rotting — a log nothing reads is a log nobody can trust. That reader is the whole of it: nothing else in the loop consults the log except `/ship`, checking the diff against it.

This matters because the harness plan file cannot be relied on — it is named after the *session*, sits in a flat global directory, is swept after 30 days, and is frequently never written. The issue survives `/clear`, `/compact`, a new session, a new machine, and a different person. It is also why `/finish` works days later in a session that knows nothing about the work.

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

Four things from those sections are **not** keystrokes, so they live in the skills: the plan-approval checklist (`/issue-to-task`), plan-conformance review (`/ship` — `/code-review` never sees the issue or the plan), the two-strike rule (`/implement`), and the definition of done (`/finish`).

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

The six-criteria checklist appears in both `/issue-to-task` and `/ship` — plan-vs-issue before the work, diff-vs-plan after. Criteria 1–5 mirror each other — the same check, worded plan-side and diff-side; change one, change the other. Criterion 6 is deliberately asymmetric: `/issue-to-task` requires the entry to be **drafted in the plan** at `Status: Proposed`, `/ship` requires it to be **written into the log** at `Status: Accepted`.

What must stay identical is the bar for what earns an entry. It now appears in three places — `/idea-to-spec`, `/issue-to-task`, `/ship` — plus `unit-of-work-process.md`. Change one, change all four.

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
| `/idea-to-spec` | Interviews you, writes `specs/<slug>.md` and an `arch/` file per decision | Overwrite an existing spec; renumber or delete an existing ADR; spec a one-line change |
| `/spec-to-issues` | A spec → ordered issues, each naming its spec, with a pass/fail check and the ADRs that bind it | Create anything before you approve the list; file an issue with no check, or with no spec |
| `/issue-to-task <n>` | Loads the issue, restates it *and the ADRs binding it*, gates the plan on six criteria, posts the plan — and any ADR it drafts — to the issue | Plan an issue whose restatement you say is wrong, or one with no verification check, until the check is written into the issue; leave the plan silent about whether it decides architecture |
| `/implement` | Branches, writes the failing tests, implements, writes the plan's ADR into `arch/`, sweeps superseded entries into `arch/archive/` | Run without an approved plan; touch a file the plan doesn't list; make a decision the plan didn't draft; archive an entry anything references by path |
| `/verify` | Runs the issue's check, shows command + full output, posts the evidence to the issue | Summarize output; proceed on a failure — a failure hands back to `/implement` |
| `/ship` | Commits first, then reviews vs. the plan and `arch/`, re-verifies, opens the PR | PR from the default branch; proceed past `/code-review` on an agent handle; open the PR without your explicit yes; use evidence older than the last fix; open a PR whose diff decides architecture with nothing recorded; merge |
| `/finish [n]` | Confirms the merge happened, closes the issue if `Closes #N` didn't fire, confirms the `arch/` files landed on the default branch, flags retired entries the sweep missed, deletes the branch | Touch anything if the PR isn't `MERGED`, or the tree is dirty; write the ADR log itself, or move anything within it |

## Where the state lives

There is no state file. The **GitHub issue is the store**: `/issue-to-task` posts the approved plan to it as a comment marked `<!-- unit-of-work:approved-plan -->`, and `/implement` and `/ship` read it back. The issue also holds `/verify`'s evidence comment, which `/finish` links for the definition of done.

The **decision log is the other store**, and the only one that outlives the issues: `arch/`, one file per decision, append-only. Specs live beside it in `specs/`, one file per feature, and every issue names the spec it came from on a `Spec:` line. `/issue-to-task` reads it back to you at restatement time on every unit of work, which is what keeps it from rotting — a log nothing reads is a log nobody can trust. Every skill in the loop touches it: `/spec-to-issues` names the gaps, `/verify` reads the consequences of the entries an issue is bound by, `/ship` checks the diff against it, and `/finish` confirms the files reached the default branch.

**`arch/archive/` is the log's cold half.** When a decision supersedes an earlier one, `/implement` moves the retired entry there as the second half of the supersede, and sweeps any earlier one left behind while it is in the neighbourhood. Nothing is ever deleted — append-only still holds, and the reversal is the most useful thing in the record — but the working set stays small enough to read. **Agents ignore the archive** with exactly two exceptions: numbering counts it, because a retired entry keeps its number and skipping it would reuse one; and `/verify` looks in it to resolve a `Constrained by ADR-<n>` on an issue older than the supersede. `/finish` flags anything retired still sitting in the top level, and never moves it — a move on the default branch is unreviewed, and it can break a build that references the entry by path.

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

The skills check the first two and tell you what's missing, but they can't fix them. Nothing checks the last two — they are on you:

1. `gh auth login` — six of the seven depend on it; only `/idea-to-spec` does not.
2. A check that runs by command. Without one, `/verify` has nothing to do.
3. `CLAUDE.md`, short — build/test/lint commands, style that differs from defaults, repo etiquette, gotchas. `/init` generates one; prune it hard.
4. `/permissions` — allowlist your test runner, linter, `git commit`, `git push`, and `gh`.

## Retrofitting an existing backlog

Issues not created by `/spec-to-issues` have no `## Verification` section. `/issue-to-task` meets one first and refuses to plan until a check is written into the issue; `/verify` proposes one, confirms it with you, and writes it in when it meets such an issue directly — fixing the issue rather than the conversation, so the next session verifies the same thing. A spec is optional everywhere except `/spec-to-issues`; an issue with no `Spec:` line is worked from the issue alone.

## Another machine

```bash
git clone git@github.com:kgrad5/skills ~/.claude/skills
```

## Editing

`SKILL.md` frontmatter is `name`, `description`, `argument-hint` only. `name` must equal its directory name.

No `allowed-tools`: it *restricts* the toolset rather than pre-approving it, and `/verify` and `/implement` must run project commands nobody can enumerate in advance. Permissions belong in `/permissions`.

The six-criteria checklist appears in three places — `/issue-to-task` and `/ship` (plan-vs-issue before the work, diff-vs-plan after), and again as the hand checklist in `unit-of-work-process.md` step 4. Criteria 1–5 mirror each other — the same check, worded plan-side and diff-side; change one, change the other. Criterion 6 is deliberately asymmetric: `/issue-to-task` requires the entry to be **drafted in the plan** at `Status: Proposed`, `/ship` requires it to be **written into the log** at `Status: Accepted`.

What must stay identical is the bar for what earns an entry. It now appears in three places — `/idea-to-spec`, `/issue-to-task`, `/ship` — plus `unit-of-work-process.md`. Change one, change all four.

**How a spec section is cited** lives in exactly one place, `/idea-to-spec` under "Citing a spec section": `specs/<slug>.md §4.4`, never a bare `§4.4`, anywhere outside the spec itself. `/spec-to-issues` and `/issue-to-task` point at it rather than restating it, because a rule copied into four files is a rule that drifts in three of them. It is also the one convention here worth enforcing with a test in the target repo — it was written down once and broken in the next file authored under it, which is what a rule that depends on memory does.

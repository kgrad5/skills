---
name: spec-to-issues
description: Breaks a spec in specs/ into an ordered list of independently implementable GitHub issues, each carrying a verification check that can pass or fail and the ADR numbers that bind it, and creates them with gh issue create after the user approves the list. This skill should be used when the user says "break the spec into issues", "create issues from SPEC.md", "turn this spec into tasks", or "file the issues".
argument-hint: [specs/<slug>.md] [--label <name>]
---

# spec-to-issues

Guarantees that every task entering the backlog is independently implementable and carries a check that can pass or fail — and that nothing gets created before the user has seen the whole list.

## Preconditions

Check and report; never assume.

- `gh auth status` succeeds. If not, stop and say to run `gh auth login`.
- The working directory is a git repository with a GitHub remote: `gh repo view --json nameWithOwner,defaultBranchRef`.
- The spec file exists. If it does not, stop and ask for the path. Never write issues from an imagined spec.
- The spec is committed: `git status --porcelain <spec-path>`. If it is dirty or untracked, stop and say so — issues must not reference a document nobody else can see.

## Inputs

Arguments given: $ARGUMENTS

- First argument not starting with `--` is the spec path. With none given, list `specs/`: use the single file if there is exactly one, and **ask** if there are several — never guess which feature the user meant. A repo predating this layout may carry a root `SPEC.md`; use it and say so.
- `--label <name>` sets the label applied to every issue. If omitted, derive a kebab-case label from the spec's feature name and confirm it with the user.

## Procedure

1. **Read the spec in full**, including its Out of scope and Verification sections, and **list `arch/`** and read the entries it holds. Skip `arch/archive/`: those decisions were retired by later ones, and an issue constrained by a decision that no longer binds is worse than an issue with no constraint named. The enumerated list is what actually gets read; a source not named here is a source skipped.
2. **Decompose it** into tasks. Each one must be:
   - **Independently implementable** — a single Claude session can plan and finish it
   - **Independently verifiable** — it has its own check, not "covered by the feature's end-to-end test"
   - **Ordered by dependency** — if task B needs task A's interface, A comes first
3. **Fold trivial tasks in.** If a task's diff is describable in one sentence, it does not deserve its own issue. Attach it to the neighboring issue that touches the same code.
4. **Draft every issue body** using the template below.
5. **Name the ADR gaps first.** Before showing anything, name every decision the spec *states* that `arch/` does not *record* — measured against the bar in `/idea-to-spec`. Report them and ask which the user wants: amend the spec now (`/idea-to-spec` in extend mode, commit, re-run this skill), or proceed — in which case write `Undocumented decision: <one sentence>` into the body of every issue the gap touches, so `/issue-to-task` restates it at step 3 and drafts the entry at plan time. A gap named only in this skill's report dies with the session. Ask; do not refuse. This skill cannot write the spec, and the spec must be committed — a hard stop here would leave no remedy inside the skill. Once step 8 has created one issue there is no safe re-run, so this is the last moment the spec is cheap to fix.

6. **Show the complete list to the user and stop.** Numbered, in dependency order, with each issue's title and its verification check. Ask for approval. Create nothing yet.
7. **Create the label first**, once approved: `gh label create <name> --force`. `gh issue create --label` fails outright on a label that does not exist, which would kill the very first issue.
8. **Create the issues one at a time, in order.** After each, echo the returned URL immediately.
9. **On any failure, stop.** Report exactly which issues were created, with their URLs and numbers, and which were not. Never restart the loop — a retry from the top creates duplicates.
10. **Record dependencies.** Where task B depends on task A, write `Depends on #<A>` into B's body once A's number is known. This is what makes the ordering enforceable later.

## The issue template

```markdown
## Goal
<One paragraph. What is true after this is done that is not true now.>

## Files likely involved
<Named paths. Best guess; the plan may revise it.>

## Out of scope
<Explicit. What this issue does not touch.>

## Verification
<One concrete check that can pass or fail. A command and its expected result.>

Spec: specs/<slug>.md             <!-- always; the spec this issue was derived from -->
Constrained by ADR-<n>, ADR-<m>   <!-- the arch/ entries this issue must obey; omit when none -->
Depends on #<n>                   <!-- only when it does -->
```

## Hard rules

- **Every issue carries a check that can pass or fail.** "curl returns 429 after the 10th request" — not "rate limiting works". If you cannot write the check, the task is not defined yet: say so and split or sharpen it rather than filing it.
- **Show the list before creating anything.** Issues are public and outward-facing. The user sees the whole set first, every time.
- **No intermediate documents.** Do not produce a PRD, a milestone doc, an epic breakdown, or a tracking issue. Spec plus issues is the whole stack. Each extra generative layer *re-derives* content that already exists somewhere else, so it drifts from intent and goes stale the moment implementation deviates. **`arch/` is the one exception:** it is append-only, it restates nothing, and superseding an entry is itself a new file — so it cannot go stale.
- **Every issue names the spec it came from.** The `Spec:` line is not decoration: `/issue-to-task`, `/implement`, `/verify` and `/ship` all read it to find the document this work answers to. A repo will hold many specs, and an issue that does not name its own sends every later skill to guess — which, once there are three specs, it will get wrong.
- **Cite sections as `specs/<slug>.md §4.4`, never a bare `§4.4`** — in the goal, the out-of-scope list and the verification check alike. The reasoning is the bullet above, applied one level down: an issue that names its spec on the `Spec:` line and then cites `§4.4` in its body has still left every quoted section ambiguous. `/idea-to-spec` states the rule in full.
- **`Constrained by` is a promise that something reads it.** `/issue-to-task` restates those entries back to the user before planning. Write the line only where an entry genuinely binds the issue; a list of every ADR is the same as no list. If the user's organization needs epics for coordination, those live in their tracker for humans and are not inputs to this process.
- **Out of scope is mandatory on every issue.** It is the boundary the implementation will be held to.
- **Never restart a partially-completed creation loop.**

## Output

Report:
- The label used.
- The spec every issue was derived from, and a numbered table: issue number, title, URL, verification check, and the ADRs each issue is constrained by.
- The ADR gaps named at step 5, and what the user chose to do about each.
- The dependency edges recorded.
- If anything failed: exactly what was created and what was not, so the user can resume by hand.

## Next step

Start the first issue in a fresh session: `/clear`, then `/issue-to-task <first-issue-number>`. One issue per session — do not begin implementing from within this skill.

---
name: brainstorm
description: Takes an idea to filed GitHub issues in one sitting — asks what the idea is, narrows it with the user to the smallest version worth building, decomposes it into single-focus steps with a dependency DAG, writes the spec and ADRs when the idea is big enough to need them, and after one approval lands them and files one issue per step with the Depends on lines /execute schedules from. Small changes take a fast path with no spec and at most one round of questions. This skill should be used when the user says "brainstorm", "I have an idea", "shape this idea", "idea to issues", or "break this idea into issues".
argument-hint: [idea] [--label <name>]
---

# brainstorm

Guarantees that an idea enters the backlog as the fewest single-focus issues that deliver it, each with a check that can pass or fail, wired into a dependency DAG that `/execute` runs in parallel — with no more ceremony than the idea's size calls for.

It is `/idea-to-spec` and `/spec-to-issues` in one pass, with one approval instead of two. Read both skills before writing anything — `~/.claude/skills/idea-to-spec/SKILL.md` and `~/.claude/skills/spec-to-issues/SKILL.md`: the spec's shape, the ADR format and the bar for an entry, section citations, the issue template and both skills' hard rules bind this skill. Where they conflict with it — idea-to-spec's one-sentence triage stop and its ban on creating issues, and the commit-then-rerun handoff between the two — this skill wins.

The full loop is two commands: `/brainstorm`, then `/execute` on the issues it files.

## Preconditions

Check and report; never assume.

- `AskUserQuestion` is available. Without it, stop: an idea shaped without the user is a guess.
- `gh auth status` succeeds, and `gh repo view --json nameWithOwner,defaultBranchRef` resolves. Use that default branch everywhere; never hardcode `main`.

## Inputs

Arguments given: $ARGUMENTS

- `--label <name>` sets the label on every issue. Otherwise use the slug of the spec the issues name, or a kebab-case name for the idea when they name none; show it at the approval.
- Everything else is the idea. With none, ask in plain text what the idea is, and wait for the answer.

## Procedure

1. **Read before asking.** Skim what the idea touches: the related code, `specs/`, the live entries in `arch/`, and `gh issue list --state open --limit 1000`. An overlapping spec is extended, never rivalled, and never overwritten without an answer; an extend never renumbers an existing `§`, which `arch/` cites and never rewrites, so a new section goes after the last one at its level. Overlapping open work becomes a dependency, not a duplicate. Use an `Explore` subagent if the repo is large.

2. **Size it, and say which in one line.**
   - **Small** — every step is clear from the code, and it fits in at most three issues. Ask at most one round of questions, and only ones whose answers change an issue or give the user's reason for a decision that meets the ADR bar. Write no spec; name an existing spec on the `Spec:` line when the change falls inside one. Put each such decision, its reason and any spec sentence it makes wrong into the issue that makes it, so that issue's plan drafts the entry (issue-to-task step 6).
   - **Full** — anything else. If a small idea later fails the small test — a step the code does not make clear, a fourth issue — switch to full and say so.

3. **Narrow it.** Propose the smallest version worth having, and what moves to Out of scope, and confirm it with `AskUserQuestion` — on the small path in the one round, and not at all when the idea is already as small as it gets. On the full path every later question is about that core: interview as idea-to-spec step 4 says — up to four questions a round, your recommendation first — skipping anything obvious or already answered, but never the out-of-scope list or the user's reason for an architecture decision. Stop when another round would change neither the issues nor the implementation the spec describes, and say so.

4. **Decompose it into steps,** one issue each. A step is:
   - **Single-focus** — one goal; one `/execute` chain plans and finishes it, and its PR reads in one sitting.
   - **Independently verifiable** — its own check that can pass or fail, never "covered by the end-to-end test".
   - **Worth an issue** — a step describable in one sentence folds into the neighbour that touches the same code.

   An edge A → B means B needs something A creates: an interface, a schema, a file. Add no other edge — each false one serializes work `/execute` would run in parallel — except between two steps that would edit the same code, which get one so their merges never conflict.

5. **Draw the DAG**: a `mermaid` `graph TD` block over the steps and every open issue a step depends on, drawn as `#<n>`, and the steps' waves — wave 1 depends on no step, wave *k* only on earlier waves. `/execute` starts each issue as soon as its own dependencies close. A cycle means two steps are one, or a shared prerequisite is missing: fix the decomposition, not the graph.

6. **Write the spec** (full path only). In a worktree outside the main checkout — `git fetch origin <default>`, then `git worktree add --detach <parent of the main checkout>/<repo>.worktrees/brainstorm-<slug> origin/<default>` — write `specs/<slug>.md` and one `arch/` file per decision that meets the bar. Number each through the reservation loop in `~/.claude/skills/execute/stage-rules.md`, as owner `brainstorm-<slug>`, starting past any appendix decision table (idea-to-spec step 2): the loop counts `arch/archive/` and creates the lock directory, so no `/execute` run, live or later, takes the same number. The main checkout is never touched, so a running `/execute` keeps its clean tree. Check the spec against idea-to-spec's hard rules and fix any miss. Then name every ADR gap — a decision an existing spec states that `arch/` does not record, in an area these steps touch — and record it now if the user has given its reason; otherwise put an `Undocumented decision:` line on the earliest issue in DAG order that it touches, and only that one, since parallel plans would each draft an entry.

7. **Draft every issue body** in spec-to-issues' template, leaving the `Depends on` numbers until step 10.

8. **One approval.** In one message: the idea in a sentence; the path taken; the spec's absolute path in the draft worktree with a line per section (on an extend, its diff); each ADR in full, since that wording lands in `arch/` as written; each ADR gap and how it is settled; that a yes lands them on `<default>` before any issue is filed; the label; the numbered steps with each one's title, check and dependencies; and the DAG. Ask with `AskUserQuestion`: approve, change something, or cancel. A change returns to the step it affects. Cancel removes the draft worktree (`git worktree remove --force`: it holds the uncommitted drafts) and its reservations. Nothing reaches the default branch or GitHub before the yes, and the yes is the acceptance event for every ADR in the set.

9. **Land the spec** (full path only). `/execute` builds every worktree from `origin/<default>`, so each issue's `Spec:` line must resolve there before the issue exists. In the draft worktree, confirm no two `arch/` files share a number and every `# ADR-<n>` heading matches its filename, and run the repo's tests that need no running service. Then commit — `Add <feature> spec and ADR-<n>, ADR-<m>` (`Extend` for an existing spec; no ADR clause when there are none) — and `git push origin HEAD:<default>`.
   - **Rejected because the branch moved:** `git fetch origin <default>`, rebase onto it, renumber any ADR that now collides through the reservation loop — filename, heading and every reference, the issue drafts and the commit message included — re-run the checks above, and push again. If the rebase stops on a conflict, resolve it and push only once the user approves the resolved hunks; otherwise `git rebase --abort` and stop, keeping the worktree.
   - **Refused by branch protection:** `git push origin HEAD:refs/heads/brainstorm-<slug>`, open a pull request from it, and ask the user to merge it in the GitHub UI; continue at step 10 once `gh pr view <pr> --json state` reads `MERGED`.

   Once the spec is on `origin/<default>`, remove the draft worktree, its reservations and any `brainstorm-<slug>` branch.

10. **File the issues** wave by wave, so every dependency's number exists before its dependant is created. Create the label only when `gh label list --search <label> --json name -q '.[].name' | grep -qxF <label>` finds none — `gh label create --force` would recolour an existing one. Then one `gh issue create --title "<title>" --label <label> --body-file -` per step, with its dependencies on a single `Depends on #<a>, #<b>` line, echoing each URL as it returns. On any failure, stop, report what was created, and print each unfiled step's full body and dependencies so it can be filed by hand; never restart the loop. Then read each filed body back (`gh issue view <n> --json body`) and confirm its `Depends on` line names exactly its parents in the DAG.

## Hard rules

- **One approval, and nothing outward before it.** No push, pull request or issue until the user says yes to the whole set.
- **Small stays small.** The fast path writes no spec and asks at most one round. Escalate the moment the idea fails the small test.
- **Every step is one issue with one focus,** a check that can pass or fail, and an explicit Out of scope.
- **The DAG is acyclic and its edges are real.** It lives in the `Depends on` lines `/execute` reads, and in this skill's report — never in a tracking issue or the spec.
- **Never invent an answer or a rationale.** What the user did not settle goes into Out of scope as an open question.

## Output

- The path taken, and why.
- The spec's path and the commit that landed it; each `arch/` file with its one-line decision.
- A table: issue number, title, URL, verification check, depends on, constrained by.
- The DAG again, with issue numbers in place of steps, and its waves.
- Each ADR gap and how it was settled; any open question.

## Next step

When every step was filed: `/clear`, then `/execute <numbers>` — the exact line, ready to paste, naming every issue filed plus every open issue they depend on, directly or through another, and saying which numbers predate this run: `/execute` never starts an issue whose open dependency it was not given. If an `/execute` started without issue numbers is still running in this repo, run nothing instead — it picks these issues up at its next re-fetch, and a second run would work them twice; if one started with numbers is working a dependency, leave that dependency out and run the line once it closes. After a failed filing, print no `/clear`: the unfiled bodies exist only in this conversation.

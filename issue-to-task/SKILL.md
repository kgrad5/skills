---
name: issue-to-task
description: Loads a GitHub issue by number, restates its goal, out-of-scope list and verification check for confirmation, enters plan mode, drafts an ADR for any architecture the plan decides, and gates the resulting plan against six criteria before posting the approved plan back to the issue. This skill should be used when the user says "work on issue 42", "let's start issue 42", "load issue 42", or "plan issue 42".
argument-hint: <issue-number>
---

# issue-to-task

Guarantees that the task being planned is the task the issue actually describes, and that the approved plan survives past this session.

## Preconditions

Check and report; never assume.

- `gh auth status` succeeds. If not, stop and say to run `gh auth login`.
- The working directory is a git repository with a GitHub remote.
- A clean starting context. If this session already worked a different task, say so and recommend `/clear` first — a clean session with a good prompt beats a long session with accumulated corrections.

## Inputs

Arguments given: $ARGUMENTS

Resolve the issue number in this order. Stop at the first that yields one.

1. The first argument, if it matches `^#?[0-9]+$`.
2. `git branch --show-current` — the first number matching `(?:^|[/-])([0-9]+)(?:-|$)`. Covers `42-rate-limit`, `feat/42-rate-limit`, `kgrad/issue-42`.
3. `gh pr view --json body -q .body` — the number in `[Cc]loses #([0-9]+)`.
4. Ask the user. Never scan the issue list for a likely candidate, and never guess.

## Procedure

1. **Read the issue**: `gh issue view <N>`, then **the spec its `Spec:` line names** — `specs/<slug>.md`. That line is authoritative: with several specs in the repo, reading the wrong one is worse than reading none. A repo predating this layout may carry a root `SPEC.md`; use it and say so. If the issue names no spec and none is obvious, say "no spec named; working from the issue alone" and continue — many repos have none.

   **The decision log is `arch/` at the repo root**, one file per decision, `arch/<NNNN>-<kebab-slug>.md`. It needs no resolving and no spec: every repo can have one. Read the entries the issue's `Constrained by ADR-<n>` line names, and where it names none, read the most recent few anyway — a repo that never went through `/spec-to-issues` carries no such line, and a log nothing reads is a log that rots. The plan names the exact `arch/` path of any entry it drafts, which is what `/implement` creates and `/ship` checks for.
2. **Check dependencies.** Scan the issue for `Depends on #<n>` or `Blocked by #<n>`. For each, run `gh issue view <n> --json state,title`. If any is still `OPEN`, warn the user with the blocking issue's number and title, and ask whether to proceed anyway. This is the only thing that makes dependency ordering mean anything.
3. **Restate, in your own words** — not by quoting the issue back:
   - The goal
   - The out-of-scope list
   - The verification check
   - **The ADR entries that bind this issue**, by number, and what each forbids. If none bind it, say so. This is the only step in the whole loop that reads the decision log back to a human, and it runs on every unit of work — a log nothing reads is a log that rots.
4. **Confirm the restatement with `AskUserQuestion`.** This is a real gate: it yields the turn to the user. Offer "That's right", "That's wrong — the issue is ambiguous", and an option to correct a specific part.
   - If the restatement is wrong, **the issue is ambiguous**. Stop. Edit the issue with `gh issue edit <N>` so it says what it means, then re-read it. Do not patch the misunderstanding in conversation — the next session will hit the same ambiguity.
   - If the issue has no verification check, stop and add one now, the same way. An issue without a check is not ready to be worked.
5. **Enter plan mode** if not already in it, using the `EnterPlanMode` tool (load it with `ToolSearch` if it is not already available). Do not assume the session is in plan mode.
6. **Explore, then plan.** Read the files the issue names and the patterns they follow. Produce a plan covering: files to change, the approach, the test plan, side effects (migrations, env vars, new dependencies), and risks. Stay inside the issue's scope.

   **Draft the ADR here, inside the plan — not afterwards.** A decision earns an entry when any of these is true: it makes a sentence in the spec wrong or incomplete (name the sentence); it chose between options whose loser would have produced a different schema, file layout, dependency, or wire format; or the issue carries an `Undocumented decision:` line, meaning the spec states a decision `arch/` never recorded and this issue is the first to implement it. In a repo with no spec, the first clause cannot fire and the second is the whole test. Neither, no entry — and when nothing qualifies, the section still gets written, holding the literal line `None — no architecture-significant decision in this plan.` Size is not the test; reversibility is.

   Cite spec sections in the entry as `specs/<slug>.md §4.4`, never a bare `§4.4` — `/idea-to-spec` states why. The wording approved here is the wording that lands in `arch/`, and `arch/` is never rewritten to fix a citation later.

   For each one, write the full entry at `**Status:** Proposed`, and add its `arch/<NNNN>-<slug>.md` path to the plan's file list. **Number from `arch/` on the default branch, not your working tree** — one past the highest file in `arch/` **and `arch/archive/` together**, which `git ls-tree -r --name-only <default-branch> arch/` gives you in one command. Counting the archive is not optional: a retired entry keeps its number, and a count that skips the archive hands the next unit of work a number already in use. Two units of work planned before either merges will reach for the same number; `/ship` re-checks and renames if another file landed first. Drafting it here rather than at step 10 is what makes approval mean something: the wording the user approves is the wording that lands on the default branch.

   If the plan would **contradict** an ADR the issue is constrained by, stop and put it to the user. That is a supersede, not a detail — the new entry supersedes the old one, and the old one is never deleted.
7. **Write the plan to the harness plan file** at the path given in the plan-mode message. This is the one write permitted in plan mode, and it makes the path available for the rest of the session.
8. **Gate the plan on the six criteria** before presenting it. State each verdict explicitly:
   - Every consequential decision is explicit — no "handle appropriately"
   - The listed files match what the issue would lead you to expect — plus an `arch/` file, when and only when this plan drafts one
   - The plan ends with the issue's verification check
   - Nothing outside the issue's scope has crept in — recording a decision *this plan makes*, or one the spec states that `arch/` never recorded and this issue implements, is inside scope; recording any other is not
   - The test plan covers the edge cases the issue names
   - **The plan states which ADR it adds, or states in as many words that it adds none.** Check the artifact, not your opinion: the Architecture decisions section exists and is non-empty. Silence fails this criterion — "I judged it insignificant" is exactly the verdict that leaves no trace

   Any criterion that fails, fix before presenting. Do not present a plan you have just described as failing.
9. **Present for approval** via `ExitPlanMode`. Tell the user they can open the plan in their editor with `Ctrl+G`, and that arguing with a paragraph is cheaper than reverting a diff. Where the plan drafts an entry, **say that the ADR text is part of what they are approving** — approval is the acceptance event, and this wording is what lands in the log.
10. **The instant the plan is approved — before any code — post it to the issue:**

    ```bash
    gh issue comment <N> --body-file - <<'EOF'
    <!-- unit-of-work:approved-plan -->
    ## Approved plan — <branch-or-issue-slug>

    ### Files
    - path/one
    - path/two

    ### Approach
    ...

    ### Test plan
    ...

    ### Architecture decisions

    arch/<NNNN>-<kebab-slug>.md
    # ADR-<n> — <noun phrase>
    **Status:** Proposed · <YYYY-MM-DD> · issue #<N>
    ... Context / Decision / Consequences / Alternatives rejected ...

    ### Architecture decisions          # ← the other form this section takes
    None — no architecture-significant decision in this plan.

    ### Verification
    <the issue's check, verbatim>
    EOF
    ```

    This is not bookkeeping. The harness plan file is named after the *session*, lives in a flat global directory, is swept after 30 days, and is often never written at all. The issue is the only place the approved file list reliably survives a `/clear`, a `/compact`, a new session, or a new machine — and `/implement` refuses to run without it.

## Hard rules

- **A wrong restatement means a broken issue, not a broken conversation.** Fix the issue.
- **No verification check, no planning.** Add one to the issue first.
- **The plan is posted to the issue before any code is written.** Every later skill reads it from there.
- **Stay inside the issue's scope.** Anything you notice but that the issue does not cover gets mentioned to the user, not planned.
- **The Architecture decisions section is never absent.** It holds entries or it holds the literal line `None — no architecture-significant decision in this plan.` An absent section is indistinguishable from a skipped judgement, and `/implement` has nothing to copy.
- **Entry numbers are monotonic and never reused** — one past the highest file across `arch/` and `arch/archive/` together. A superseded entry moves to the archive but keeps its number forever, so the archive is part of the count even though nothing else reads it. The filename is `<NNNN>-<kebab-slug>.md`, zero-padded to four digits so the directory sorts in decision order. **Fields you do not have are omitted, not filled.**

## Output

Report the issue number and title, your restatement and its confirmation, the ADRs that bind the issue, any open dependencies, the six-criteria verdicts, any ADR drafted and the `arch/` path it will land at, and the URL of the posted plan comment.

## Next step

Run `/implement` — it will create the branch and work from the plan comment you just posted.

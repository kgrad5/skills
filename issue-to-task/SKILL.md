---
name: issue-to-task
description: Loads a GitHub issue by number, restates its goal, out-of-scope list and verification check for confirmation, enters plan mode, and gates the resulting plan against five criteria before posting the approved plan back to the issue. This skill should be used when the user says "work on issue 42", "let's start issue 42", "load issue 42", or "plan issue 42".
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

1. **Read the issue**: `gh issue view <N>`. Read `./SPEC.md` if it exists. If it does not, say "no SPEC.md; working from the issue alone" and continue — most repos have none.
2. **Check dependencies.** Scan the issue for `Depends on #<n>` or `Blocked by #<n>`. For each, run `gh issue view <n> --json state,title`. If any is still `OPEN`, warn the user with the blocking issue's number and title, and ask whether to proceed anyway. This is the only thing that makes dependency ordering mean anything.
3. **Restate, in your own words** — not by quoting the issue back:
   - The goal
   - The out-of-scope list
   - The verification check
4. **Confirm the restatement with `AskUserQuestion`.** This is a real gate: it yields the turn to the user. Offer "That's right", "That's wrong — the issue is ambiguous", and an option to correct a specific part.
   - If the restatement is wrong, **the issue is ambiguous**. Stop. Edit the issue with `gh issue edit <N>` so it says what it means, then re-read it. Do not patch the misunderstanding in conversation — the next session will hit the same ambiguity.
   - If the issue has no verification check, stop and add one now, the same way. An issue without a check is not ready to be worked.
5. **Enter plan mode** if not already in it, using the `EnterPlanMode` tool (load it with `ToolSearch` if it is not already available). Do not assume the session is in plan mode.
6. **Explore, then plan.** Read the files the issue names and the patterns they follow. Produce a plan covering: files to change, the approach, the test plan, side effects (migrations, env vars, new dependencies), and risks. Stay inside the issue's scope.
7. **Write the plan to the harness plan file** at the path given in the plan-mode message. This is the one write permitted in plan mode, and it makes the path available for the rest of the session.
8. **Gate the plan on the five criteria** before presenting it. State each verdict explicitly:
   - Every consequential decision is explicit — no "handle appropriately"
   - The listed files match what the issue would lead you to expect
   - The plan ends with the issue's verification check
   - Nothing outside the issue's scope has crept in
   - The test plan covers the edge cases the issue names

   Any criterion that fails, fix before presenting. Do not present a plan you have just described as failing.
9. **Present for approval** via `ExitPlanMode`. Tell the user they can open the plan in their editor with `Ctrl+G`, and that arguing with a paragraph is cheaper than reverting a diff.
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

## Output

Report the issue number and title, your restatement and its confirmation, any open dependencies, the five-criteria verdicts, and the URL of the posted plan comment.

## Next step

Run `/implement` — it will create the branch and work from the plan comment you just posted.

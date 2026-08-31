---
name: verify
description: Runs the verification check defined in a GitHub issue and shows the exact command and its full output as evidence, adding a check to the issue first if it has none, and flagging any ADR consequence the evidence contradicts. This skill should be used when the user says "verify it", "run the check", "prove it works", or "verify issue 42".
argument-hint: [issue-number]
---

# verify

Guarantees that "done" rests on evidence rather than assertion.

## Preconditions

Check and report; never assume.

- `gh auth status` succeeds.
- A test or check command is discoverable from `CLAUDE.md`, `package.json`, `Makefile`, `pyproject.toml`, or the issue itself. If none is found, **ask**. Never invent one.
- **The spec this issue descends from is named on its `Spec:` line** — `specs/<slug>.md`. Read that file, not whichever spec happens to be at hand: a check written against the wrong spec proves nothing about this one. A repo predating the `specs/` layout may carry a root `SPEC.md`; use it and say so. If the issue names **no** spec, say so and carry on from the issue alone — a spec is optional everywhere in this loop. Only stop when the issue names a spec the repo does not hold.
- If the issue carries a `Constrained by ADR-<n>` line, the matching file exists in `arch/` **or in `arch/archive/`**. An issue outlives the entry that bound it, so look in both before concluding anything; only when neither holds it do you say so and stop. Never reconstruct the decision from the issue. When the match comes from the archive, read it and say so, naming the entry that superseded it — the constraint the issue was written against is no longer the live one, and that is worth knowing before the check runs rather than after.

## Inputs

Arguments given: $ARGUMENTS

Resolve the issue number in this order. Stop at the first that yields one.

1. The first argument, if it matches `^#?[0-9]+$`.
2. `git branch --show-current` — the first number matching `(?:^|[/-])([0-9]+)(?:-|$)`. Covers `42-rate-limit`, `feat/42-rate-limit`, `kgrad/issue-42`.
3. `gh pr view --json body -q .body` — the number in `[Cc]loses #([0-9]+)`.
4. Ask the user. Never scan the issue list for a likely candidate, and never guess.

## Procedure

1. **Read the issue's Verification section**: `gh issue view <N>`, and the spec its `Spec:` line names. Read the **Consequences** of every ADR the issue names, before running the check — you cannot notice a contradiction with text you have not read.
2. **If the issue has no verification check** — the normal case for any pre-existing backlog:
   - Propose one concrete check that can pass or fail, derived from the issue's goal and the code as it now stands.
   - Confirm it with `AskUserQuestion`: use the proposal, edit it, or take one the user supplies.
   - **Write it into the issue** with `gh issue edit <N>`, under a `## Verification` heading.
   - Then run it. Fixing the issue rather than the conversation is what lets the next session — and the next person — verify the same thing.
3. **Run the check.** Show the exact command and its complete output, verbatim. Do not summarize, truncate, or paraphrase it. Reviewing real output is faster than re-running the check.
4. **If the check fails:** print the failure verbatim, state plainly that the task is not done, and stop. Do not fix it here — hand back to `/implement`.
5. **If the check cannot be run in this session** — it needs a browser, a staging environment, credentials you do not have — say exactly what blocks it, ask the user to run it and paste the output, and treat that output as the evidence. "I cannot run this" is not the same as "this is not done."
6. **Post the evidence to the issue** as a comment, so the record is timestamped and survives this session. If the output contradicts a consequence an ADR claims, post that in the same comment: the entry's number, the consequence line, and the output line that refutes it. Do not amend the log here — a contradiction is the next unit of work, and this skill's job is to make it undeniable, not to resolve it. A contradiction that lives only in this conversation is gone by the next session, which is the same reason the check itself gets written into the issue.

## Hard rules

- **Evidence, not assertion.** "Tests pass" is not evidence. The command and its output are.
- **Verbatim output.** No summarizing, no eliding, no "(truncated)". If it is long, it is long.
- **No check, no done.** There is no exception. If the check cannot be defined, define one and write it into the issue before proceeding.
- **A failing check ends the run.** Do not proceed to review or shipping on a failure, and do not fix the failure from inside this skill.

## Output

- The verification check, quoted from the issue.
- The exact command run.
- Its full output.
- A one-line verdict: passed, failed, or blocked — and if blocked, exactly what blocks it.
- Any ADR consequence the evidence contradicts, quoted against the output line that refutes it.
- The URL of the evidence comment.

## Next step

On a pass, run `/ship` — it will review the diff against the plan, commit, and open the PR. On a failure, return to `/implement`.

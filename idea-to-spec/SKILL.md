---
name: idea-to-spec
description: Interviews the user in depth about something they want to build, then writes a self-contained SPEC.md that names the files and interfaces involved, states what is out of scope, and ends with an end-to-end verification step. This skill should be used when the user says "I want to build", "interview me", "spec this out", "write a spec", or "turn this idea into a spec".
argument-hint: <what to build> [path/to/SPEC.md]
---

# idea-to-spec

Guarantees that a feature is fully interrogated before any code exists, and that what comes out is a spec someone else could implement from without asking follow-up questions.

## Preconditions

Check and report; never assume.

- This skill depends on `AskUserQuestion`, which is unavailable in non-interactive runs (`claude -p`). If the tool is not available, say so plainly and stop — a spec written without the interview is a guess.
- The working directory is writable.

## Inputs

Arguments given: $ARGUMENTS

- A trailing argument ending in `.md` is the output path. Otherwise the output is `./SPEC.md`.
- Everything else is the description of what to build.
- If no description was given, ask what to build before doing anything else.

## Procedure

1. **Triage before interviewing.** If the whole thing is describable in one sentence — a typo, a log line, a renamed variable — say so and stop. Tell the user to ask for the change directly instead. A spec for a one-line diff is waste, and so is the process that follows it.
2. **Check for an existing spec.** If the output path already exists, read it, summarize what it covers, and ask whether to extend or replace it. Never overwrite without an explicit answer.
3. **Read before asking.** Skim the repo for what already exists in this area — related modules, the patterns they follow, the test layout. Questions grounded in real files are worth several generic ones. Use subagents for this if the codebase is large, so file contents do not flood the conversation.
4. **Interview with `AskUserQuestion`.** Multiple rounds, up to four questions each. Cover, at minimum:
   - **Technical implementation** — data model, interfaces, where this lives relative to existing code, what it depends on
   - **UI/UX** — what the user sees, what they do, what happens when they do the wrong thing
   - **Edge cases** — empty, huge, concurrent, offline, unauthorized, already-exists, partially-failed
   - **Concerns and tradeoffs** — what could go wrong, what gets slower, what gets harder to change later
   - **Out of scope** — what this feature is explicitly not, and what is deferred to later work

   Skip anything whose answer is obvious or already given. Spend the questions on the parts the user probably has not considered. Where you have a recommendation, make it the first option and mark it recommended.
5. **Keep going until it is covered.** Do not stop at one round. Stop when a new round would only produce questions whose answers do not change the implementation. State that you have reached that point before writing.
6. **Write the spec** to the output path.
7. **Check it against the quality bar below.** Report which criteria it meets. If one fails, fix it before handing back.
8. **Tell the user to review and commit it.** The spec is the input to every issue that follows; an uncommitted spec is one nobody else can see.

## The spec's shape

```markdown
# <Feature>

## Goal
<One paragraph. What this makes possible that is not possible today.>

## Approach
<How it works. The data model, the interfaces, the flow.>

## Files and interfaces
<Named paths and signatures. Not "the auth layer" — `src/auth/session.py`, `create_session(user_id) -> Session`.>

## Edge cases
<Each one named, with the intended behavior.>

## Out of scope
<Explicit. What this does not do, and what is deferred.>

## Verification
<An end-to-end check that proves the feature works. A command and its expected result.>
```

## Hard rules

- **Self-contained.** A reader with the repo and the spec, and no access to this conversation, must be able to implement it. Every decision reached in the interview belongs in the document, not in the scrollback.
- **Named files and interfaces.** Not areas or layers. Paths and signatures.
- **Out of scope is mandatory and explicit.** A spec that does not say what it excludes will be over-implemented.
- **It ends with a verification step that can pass or fail.** "The feature works" is not one. "`curl -s localhost:8000/health` returns `200`" is.
- **Never invent an answer the user did not give.** If something was not covered, ask, or write it into Out of scope as an open question. Do not fill the gap with a plausible default and present it as decided.
- **Do not create PRDs, milestone docs, or epic breakdowns.** Spec plus issues is the whole stack. Every extra generative layer drifts from intent and goes stale the moment implementation deviates.

## Output

Report:
- The path written.
- A one-line summary of each section.
- Which quality-bar criteria the spec meets, and the verification step verbatim.
- Anything left as an open question.

## Next step

Have the user review and commit the spec, then run `/spec-to-issues` to break it into issues. Do not create issues from within this skill.

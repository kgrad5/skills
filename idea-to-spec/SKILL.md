---
name: idea-to-spec
description: Interviews the user in depth about something they want to build, then writes a self-contained spec to specs/ that names the files and interfaces involved, states what is out of scope, records every architecture decision as its own file in arch/, and ends with an end-to-end verification step. This skill should be used when the user says "I want to build", "interview me", "spec this out", "write a spec", or "turn this idea into a spec".
argument-hint: <what to build> [specs/<slug>.md]
---

# idea-to-spec

Guarantees that a feature is fully interrogated before any code exists, and that what comes out is a spec someone else could implement from without asking follow-up questions.

## Preconditions

Check and report; never assume.

- This skill depends on `AskUserQuestion`, which is unavailable in non-interactive runs (`claude -p`). If the tool is not available, say so plainly and stop — a spec written without the interview is a guess.
- The working directory is writable.

## Inputs

Arguments given: $ARGUMENTS

- A trailing argument ending in `.md` is the output path. Otherwise the output is `specs/<kebab-slug>.md`, the slug taken from the feature name — one file per spec, so a repo accumulates specs rather than overwriting one. A repo predating this layout may carry a root `SPEC.md`; use it and say so.
- Everything else is the description of what to build.
- If no description was given, ask what to build before doing anything else.

## Procedure

1. **Triage before interviewing.** If the whole thing is describable in one sentence — a typo, a log line, a renamed variable — say so and stop. Tell the user to ask for the change directly instead. A spec for a one-line diff is waste, and so is the process that follows it.
2. **Check for an existing spec.** List `specs/` and say what is already there — a feature that overlaps an existing spec should extend it rather than start a rival. If the output path already exists, read it, summarize what it covers, and ask whether to extend or replace it. Never overwrite without an explicit answer.

   **A replace rewrites the spec, never `arch/`.** Those files are a separate, append-only record: issues, pull requests and plan comments cite their numbers, and a replaced spec that orphans ADR-13 leaves a `Constrained by ADR-13` line pointing at nothing. A decision the new spec reverses gets a *new* file that supersedes the old one. Delete nothing — a replace rewrites what the system does, not the record of how it got that way.

   If the spec carries an older decision *table* in an appendix, leave it exactly where it sits and start `arch/` numbering one past its highest row. **Do not convert the table into files** — rewriting rationales you were not in the room for invents rationales.
3. **Read before asking.** Skim the repo for what already exists in this area — related modules, the patterns they follow, the test layout. Questions grounded in real files are worth several generic ones. Use subagents for this if the codebase is large, so file contents do not flood the conversation.
4. **Interview with `AskUserQuestion`.** Multiple rounds, up to four questions each. Cover, at minimum:
   - **Technical implementation** — data model, interfaces, where this lives relative to existing code, what it depends on
   - **UI/UX** — what the user sees, what they do, what happens when they do the wrong thing
   - **Edge cases** — empty, huge, concurrent, offline, unauthorized, already-exists, partially-failed
   - **Concerns and tradeoffs** — what could go wrong, what gets slower, what gets harder to change later
   - **Out of scope** — what this feature is explicitly not, and what is deferred to later work
   - **Architecture decisions** — for each fork that meets the bar below, what the alternative was and *why it loses*. You need the user's reason, not a plausible one; the rationale is the whole value of the record and the only part nobody can reconstruct later

   Skip anything whose answer is obvious or already given. Spend the questions on the parts the user probably has not considered. Where you have a recommendation, make it the first option and mark it recommended.
5. **Keep going until it is covered.** Do not stop at one round. Stop when a new round would only produce questions whose answers do not change the implementation. State that you have reached that point before writing.
6. **Write the spec** to the output path, and **one file per architecture decision into `arch/`**. Every decision the interview settled that meets the bar below gets a file. On a new repo they are numbered from 1; otherwise they continue from one past the highest number already in `arch/`, per step 2. The spec itself carries only a pointer to the directory — `ls arch/` is the index, so nothing has to be kept in sync twice.
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

## Decisions

One file per decision in [`arch/`](arch/). `ls arch/` is the index.
```

Each decision is its own file, `arch/<NNNN>-<kebab-slug>.md`, zero-padded to four digits so the
directory sorts in decision order:

```markdown
# ADR-1 — <the decision, as a noun phrase>

**Status:** Accepted · <YYYY-MM-DD> · spec
**Supersedes:** ADR-<m>          <!-- omit this line entirely when it supersedes nothing -->

**Context**
<The forces. What made this a decision rather than an obvious call.>

**Decision**
<What was chosen. Active voice.>

**Consequences**
- <What this makes easy or safe.>
- <What it costs, and what it forecloses.>

**Alternatives rejected**
- <Option> — <why it lost>
```

### Citing a spec section

**Every reference to a spec section names the spec file — `specs/<slug>.md §4.4`, never a bare
`§4.4`.** This holds in ADR bodies, issue bodies, plans, commit messages, PR bodies and code
comments: anywhere outside the spec itself. Naming it once per line is enough, so a later `§` on
the same line reads against it.

A bare `§4.4` is unambiguous only while a repo holds exactly one spec, and the `specs/` layout
exists precisely because it will hold several. A reader two specs later cannot recover which
document was meant, and `arch/` — the place these references cluster — is the artifact that is
never rewritten to fix them.

Inside `specs/<slug>.md` itself a bare `§` is correct: a document citing its own sections.

**Enforce this rather than remember it.** A test that fails on a `§` appearing on a line with no
`specs/<slug>.md` costs a dozen lines, and grandfathering the entries that predate it by name
keeps `arch/` append-only while stopping new ones. Written down and not enforced, this rule
survives about one file.

### What earns an entry

A decision earns an ADR when either of these is true:

- It makes a sentence in the spec wrong or incomplete. Name the sentence.
- It chose between options whose **loser would have produced a different schema, file layout, dependency, or wire format**.

Neither, no entry. **Size is not the test; reversibility is.** A one-line change that pins a runtime or adds a dependency earns an entry. A hundred-line refactor that changes nothing anyone else must obey does not. A log is worth reading only if it stayed short.

A spec-born ADR carries `spec` where an issue number would go: at this point no issue and no pull request exist, and this skill is forbidden from creating them. The user's approval at step 8 is its acceptance event — an entry they reject never lands.

## Hard rules

- **Self-contained.** A reader with the repo and the spec, and no access to this conversation, must be able to implement it. Every decision reached in the interview belongs in the document, not in the scrollback.
- **Named files and interfaces.** Not areas or layers. Paths and signatures.
- **Out of scope is mandatory and explicit.** A spec that does not say what it excludes will be over-implemented.
- **It ends with a verification step that can pass or fail.** "The feature works" is not one. "`curl -s localhost:8000/health` returns `200`" is.
- **Never invent an answer the user did not give.** If something was not covered, ask, or write it into Out of scope as an open question. Do not fill the gap with a plausible default and present it as decided.
- **Do not create PRDs, milestone docs, or epic breakdowns.** Spec plus issues is the whole stack. Every extra generative layer *re-derives* content that already exists somewhere else, so it drifts from intent and goes stale the moment implementation deviates. **`arch/` is the one exception, and not because of where it sits.** It is append-only, it restates nothing — each file is the sole record of a decision and the options it closed off — and it cannot go stale, because superseding an entry is itself a new file. If you catch yourself copying the Approach section into an ADR, you have built the layer this rule forbids.
- **An ADR's Context, Consequences and Alternatives come from the interview.** If the user chose without saying why, ask. **Fields you do not have are omitted, not filled** — an entry that is a decision and one line of context is a complete entry. An invented rationale is worse than a missing one: it is the sentence that stops someone reopening a decision that deserved reopening.

## Output

Report:
- The path written.
- A one-line summary of each section.
- Which quality-bar criteria the spec meets, and the verification step verbatim.
- The `arch/` files written, each with its one-line decision — and, on an extend or replace, that no pre-existing file was renumbered or removed.
- Anything left as an open question.

## Next step

Have the user review and commit the spec, then run `/spec-to-issues` to break it into issues. Do not create issues from within this skill.

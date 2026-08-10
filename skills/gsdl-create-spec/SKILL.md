---
name: gsdl-create-spec
description: Generate SPEC.md from a seed file or user input. Use when the user wants to produce a plan, write a spec, document requirements, formalize a feature, or turn an idea/ticket/doc into a structured plan. This is Step 3 (Produce a Plan) of the GSDL pipeline — its output IS the checkpoint. Runs on the LARGE tier.
tier: large
disable-model-invocation: true
---

# Create SPEC.md

Generates `SPEC.md` — the **Step 3 checkpoint** of the GSDL pipeline:
`Pick up source → Brainstorm → SPEC.md → Slices → Review → Branch → Execute → Verify → PR + bot → Close loop`

`SPEC.md` is the single, canonical plan document for the project. It is designed to be clear and
actionable enough for a junior developer — or an execution subagent — to implement from, and
detailed enough for a human reviewer to approve or reject at the Step 5 checkpoint.

## Tier

**LARGE.** This step absorbs Step 2 (brainstorm / clarify intent): asking the questions that
actually matter, surfacing tradeoffs the source didn't consider, and turning fuzzy intent into a
plan every downstream MEDIUM subagent implements against. Errors here propagate through the whole
pipeline.

The `gsdl` orchestrator can be invoked from any session model, and routes this step to LARGE either
way: **inline** when the session is already on a LARGE model (preferred — the Q&A is direct),
otherwise in a **LARGE subagent** driven by the orchestrator's Question Relay protocol, which puts
the clarifying questions to the user and feeds the answers back.

**Never run this step on a smaller tier**, and never ask the user to switch models themselves —
that's the orchestrator's job. See the `gsdl` skill's `models.md` for the tier mapping.

## Prerequisites

1. **Project exists**: `.planning/[project-name]/`
2. **Seed file exists**: `.planning/[project-name]/seed.md` — the Step 1 output

If the project structure doesn't exist, run `gsdl-fetch-source` first. It accepts a tracker item, a
document, or a plain idea, so there is always a way in.

## Process

### Step 1: Read the Seed File from Disk

Read `.planning/[project-name]/seed.md` **directly from disk** — never rely on an in-context version.
The user may have edited it since setup.

Use it to understand: the problem, rough feature ideas, references, and open questions. Note its
`## Source` block — a spec built from a tracked item should honour that item's acceptance criteria;
a spec built from a plain idea has more room to shape scope with the user.

### Step 2: Ask Clarifying Questions

**Do not skip this step** — it is the tail end of Step 2 (Brainstorm/Clarify Intent). Ask enough
questions to remove ambiguity before writing anything.

Adapt questions to the seed content:

| Area | Example Questions |
|------|-------------------|
| **Problem/Goal** | "What problem does this solve?" / "What is the main goal?" |
| **Target User** | "Who is the primary user?" |
| **Core Functionality** | "What are the key actions a user should be able to perform?" |
| **User Stories** | "As a [user], I want to [action] so that [benefit]" |
| **Acceptance Criteria** | "How will we know this is successfully implemented?" |
| **Scope/Boundaries** | "What should this explicitly not do (non-goals)?" |
| **Data Requirements** | "What data does this need to display or manipulate?" |
| **Design/UI** | "Are there mockups or UI guidelines to follow?" |
| **Edge Cases** | "What edge cases or error conditions matter?" |
| **Verification** | "How will this be verified — existing test suite, new tests, manual QA gate?" (feeds Step 7) |

Focus on the "what" and "why" — the developer/subagent figures out the "how."

See [`examples.md`](./examples.md) for worked question sets and the specs they produced.

### Step 3: Generate SPEC.md

```markdown
# [Project Name] — SPEC

## 1. Introduction/Overview
Briefly describe the feature and the problem it solves. State the goal.

## 2. Goals
- Goal 1
- Goal 2

## 3. User Stories
- As a [type of user], I want to [action] so that [benefit]

## 4. Functional Requirements
1. The system must allow users to...
2. The system must validate...
3. The system must display...

## 5. Non-Goals (Out of Scope)
- Will not support...
- Future consideration: ...

## 6. Design Considerations (Optional)
Link to mockups, describe UI/UX requirements, or relevant components/styles.

## 7. Technical Considerations (Optional)
- Should integrate with...
- Consider using...

## 8. Verification Plan
How will this be verified at Step 7 (deterministic gates)? e.g. "existing `npm test` + `npm run lint`", "new unit tests for X", "manual QA: steps to reproduce".

## 9. Success Metrics
- Metric 1: ...
- Metric 2: ...

## 10. Open Questions
- [ ] Question 1
- [ ] Question 2
```

### Step 4: Save SPEC.md

1. Save to: `.planning/[project-name]/SPEC.md` — always this exact filename, never per-feature variants
2. Confirm creation with the user, showing the file path
3. This is the **checkpoint** — do not silently proceed to decomposition. Show it to the user for
   review before Step 4 (Decompose) begins.

## Target Audience

The reader is a **junior developer or an execution subagent**. Requirements should be:
- Explicit and unambiguous
- Free of unexplained jargon
- Detailed enough to understand purpose and core logic without asking follow-up questions

## Important Rules

1. **Do NOT start implementing** after creating `SPEC.md`
2. **Always ask clarifying questions** before generating (Step 2 is not optional)
3. **Iterate on `SPEC.md`** based on user feedback — this file is re-read from disk by every later
   step, so edits are always picked up
4. **One `SPEC.md` per project** — if the project covers multiple features, use `##` sections within
   the single file rather than multiple spec files, so decomposition (Step 4) always has one source
   of truth
5. **The Verification Plan is not optional** — Step 7 discovers gates from the repo, but this section
   is where the user's own expectations get recorded

## Edge Cases

### No Seed File
Run `gsdl-fetch-source` first rather than improvising a description — it handles the no-tracker case
too, so there is no reason to skip it.

### Vague Requirements
Ask more specific questions, suggest concrete options, break broad concepts into smaller specifics.

### Multiple Features
Use separate `##` sections within the one `SPEC.md`, each with its own goals/requirements/non-goals
— decomposition will turn each section into its own slice(s).

### Existing SPEC.md
- Default to **updating it in place** and noting the change in an "## Amendments" section with a date
- Only create a dated snapshot (`SPEC-YYYYMMDD.md`) if the user explicitly wants to preserve the
  prior version before a major rewrite

## Output

- **Format:** Markdown (`.md`)
- **Location:** `.planning/[project-name]/`
- **Filename:** `SPEC.md` (always — not `prd-*.md` or feature-specific names)
- **Example:** `.planning/eng-441-user-authentication/SPEC.md`

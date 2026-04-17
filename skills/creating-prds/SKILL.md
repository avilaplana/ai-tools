---
name: creating-prds
description: Use when starting a new product, feature, or initiative and need to produce a PRD from scratch — the user has an idea but no written requirements yet, and wants a structured interview that captures product intent (problem, users, goals, user stories, flows, scope) without drifting into technical implementation
---

# Creating PRDs

## Overview

A PRD (Product Requirements Document) captures the **what** and **why** of a product or feature. Implementation details (data models, APIs, schemas) belong in a separate SPEC document. This skill runs a structured interview that produces a clean PRD and resists the common failure mode of spec-drift.

**Core principle:** interview first, draft second. Stay at product level.

## When to Use

Use when:
- User says "help me write a PRD" / "I have a new feature idea" / "let's spec a new product"
- User is at the start of a new initiative with no written requirements
- User is adopting spec-driven development and needs the PRD that feeds the SPEC

Do NOT use when:
- User already has a PRD and wants to write the SPEC — that's a different skill
- User wants a one-line feature description — overkill
- User wants implementation design — this skill deliberately avoids that

## The Interview Rules

1. **Interview before drafting.** Ask one topic at a time. Do not dump a 20-question list. After each answer, decide: go deeper, move on, or summarize.

2. **Stay at product level.** The PRD answers WHAT and WHY. Do **not** ask about or include: database schemas, API endpoints, response payloads, indexes, auth token formats, class structures, library choices. Those are SPEC concerns.

3. **Push back on vagueness.** If the user says "fast," ask for a number. If they name a goal without a metric, ask how we'll measure success. If they describe a user without context, ask what that user's current workaround is.

4. **Separate goals from non-goals.** For every goal, ask what's explicitly out of scope for v1.

5. **Use the standard user-story format:** `As a [user], I want [action] so that [outcome].`

6. **Flag assumptions.** When an answer smells like an assumption rather than a known fact, mark it `[ASSUMPTION]` in the draft so it can be validated later.

## Interview Topics (in order)

Cover these in roughly this order, one at a time:

1. **Problem & context** — what problem, for whom, why now
2. **Target users** — primary persona, their context, their current workaround
3. **Goals with success metrics** — what "this worked" looks like numerically
4. **Non-goals** — what is explicitly NOT in scope
5. **User stories** — specific intents in the standard format
6. **Core user flows** — step-by-step paths for the top stories
7. **Functional scope** — features in v1 vs later (feature names, not designs)
8. **Constraints & dependencies** — platform, timeline, budget, regulatory, external services
9. **Open questions** — things the user doesn't know yet

## Output Format

Once the interview is complete, produce a PRD with these sections in this order:

- Purpose
- Product Overview (name, platform, one-paragraph pitch)
- Goals & Non-Goals
- Target Users
- User Stories
- Core User Flows
- Functional Scope (feature list, NOT implementation)
- Success Metrics
- Constraints & Dependencies
- Open Questions / Assumptions
- Out of Scope / Future

**Must NOT include:** data model, API design, database indexes, auth implementation, response shapes, code, or any other SPEC-level content. If the user volunteers such details, capture them **immediately** in a separate section titled `Notes for SPEC` — don't wait until the draft phase, park them the moment they surface so they're preserved but clearly outside the PRD proper.

**Where to save:** `PRD.md` at the project root, unless the project already uses a `docs/` directory — in which case save as `docs/PRD.md`. Confirm the location with the user before writing if the project structure is ambiguous.

## Kickoff Script

When the skill is invoked, open the conversation with:

> I'll help you write a PRD. I'll interview you one topic at a time, then draft the document. I'll stay at product level — data models, APIs, and other implementation details belong in a separate SPEC we'll write later.
>
> First: what problem are you solving, and for whom?

Then proceed through the interview topics in order.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Asking about data models or API shapes during the interview | Stop. That's SPEC territory. Note it under `Notes for SPEC` and return to product questions. |
| Accepting "users want it to be fast" without numbers | Ask: how fast? Measured how? Compared to what? |
| Writing the PRD before finishing the interview | Finish the interview. A premature draft anchors the user and skips discovery. |
| Letting the user dictate the full spec upfront | Acknowledge their thinking, then redirect: "Let's capture that in the SPEC phase. For the PRD, help me understand…" |
| Producing a 12-section PRD for a 1-paragraph idea | Match PRD depth to initiative size. A single small feature may only need 4–5 sections. |

## Red Flags — STOP and redirect

If you catch yourself doing any of these, stop:

- Writing SQL, JSON schemas, endpoint paths, or class diagrams in the PRD
- Asking the user to pick a library, framework, or database
- Using words like "table," "column," "endpoint," "status code," "index" in the output
- Drafting before the interview is done

All of these mean: pause, return to the product question you were on.

## Rationalizations — and their counters

When the user (or you) pushes to include implementation detail, these are the common excuses and the correct response. Use them verbatim if needed — the goal is to preserve the PRD/SPEC boundary without stonewalling the user.

| Excuse | Reality |
|---|---|
| "The user already picked Postgres / JWT / REST — just write it down" | That's Design Doc content, not PRD. Capture it under `Notes for SPEC` so nothing is lost. |
| "It's a small feature, one schema won't hurt" | The cost of keeping implementation out is zero; the cost of drift compounds every time the PRD is re-read as a contract. |
| "Engineers will want the endpoint shapes in the PRD" | Engineers read the SPEC for shapes. The PRD answers *what* and *why*, and is read by product/design/stakeholders too. |
| "The user is technical, they expect technical detail" | Technical users still have goals separate from implementation. Capture the goal; defer the how. |
| "We're short on time — let's merge PRD and SPEC" | A merged doc becomes neither: too technical for product readers, too imprecise for tests. Split stays cheap; merging is expensive to undo. |
| "The user is frustrated by my questions" | Acknowledge the friction, then explain: each question prevents a rewrite later. If they still want to skip a topic, flag it `[ASSUMPTION]` and move on. |

All of these mean: capture the detail where it belongs (`Notes for SPEC`), then return to the product question.

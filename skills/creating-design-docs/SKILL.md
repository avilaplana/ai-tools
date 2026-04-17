---
name: creating-design-docs
description: Use when a PRD is complete and the user is about to start technical work — writing SPECs, picking a stack, or locking in cross-cutting decisions like architecture pattern, service topology, shared data model, API conventions, async messaging, auth approach, or deployment. Also use when the user asks about these concerns in a way that spans multiple features or services rather than one. If a design doc already exists, use this skill to produce an ADR (Architecture Decision Record) rather than overwriting.
---

# Creating Design Docs

## Overview

A Design Doc captures the **what approach** and **why this way** for cross-cutting technical decisions — the choices that span multiple features or services (tech stack, architecture, shared entities, API conventions, messaging, auth, deployment). It sits between the PRD (what/why) and feature SPECs (how, precisely, per feature). Without it, either each SPEC re-decides these concerns in conflict with the others, or the first SPEC locks them in implicitly for everyone.

**Core principle:** cross-cutting only. **Anything that applies across features/services → design doc. Anything per-feature → SPEC.**

## When to Use

Use when:
- User has a PRD and is starting technical work (before any feature SPEC)
- User asks about tech stack, architecture, shared data model, API conventions, messaging, auth, or deployment
- Multiple SPECs would otherwise need to reinvent the same decision

Do NOT use when:
- User wants a feature SPEC (per-feature HOW) — different skill
- User wants a PRD (product what/why) — use `creating-prds`
- User wants code — the design doc isn't code, and you don't code from it directly

## When a design doc already exists

Before starting the interview, check whether `docs/design.md` (or the project's equivalent) is already present. If it is, **do NOT overwrite it** — the interview would produce conflicting decisions and destroy institutional history. Instead, produce an **ADR (Architecture Decision Record)**:

1. Read the existing design doc to understand what's already settled.
2. Ask the user what's changing: a new cross-cutting decision, a reversal of an earlier one, or a scope extension (e.g., adding microservices to a monolith).
3. Write the ADR to `docs/adr/NNNN-<short-title>.md` (where `NNNN` is the next sequential number) with this shape:
   - **Status:** Proposed / Accepted / Superseded
   - **Context:** what changed since the design doc was written
   - **Decision:** the new cross-cutting choice and its rationale
   - **Consequences:** what this affects downstream (which SPECs, which services, which conventions)
   - **Supersedes:** link to the section of the design doc this replaces (if any)

The ADR becomes the source of truth for that decision. The original design doc stays intact as historical context. Future SPEC authors read both.

## Two Sizes — choose before writing

Ask the user which they need. Default to lightweight unless something below forces full RFC.

| Size | When | What's in it |
|---|---|---|
| **Lightweight architecture doc** (~300–800 words) | Solo / v1 / small team; choices are familiar or cheap to reverse | Each cross-cutting decision stated + one-line rationale |
| **Full RFC** (~1,500–4,000 words) | Contested approach; new service; cross-team impact; expensive to reverse | Above + alternatives analysis, trade-off matrix, risks & mitigations, reviewer sign-off |

If the user asks for a "comprehensive" or "rigorous" doc for a solo/weekend project, push back: ceremony should match stakes. A 3,000-word RFC for 200 lines of code is displacement activity.

**Interview depth scales with size:**

- **Lightweight:** 3–4 batched rounds, clustering related topics per round (e.g., stack + architecture + entities in one round; conventions + auth + deployment in another). Prefer `[ASSUMPTION]` flags over probing. Target ~10 minutes.
- **Full RFC:** one topic per round with follow-ups. Probe alternatives, trade-offs, failure modes. Target 30+ minutes.

Over-interviewing a lightweight doc is itself displacement activity. If you're on question 15 of a lightweight interview, stop and draft.

## The Interview Rules

1. **Interview before drafting.** Pace to the size: in Full RFC mode, one topic at a time with follow-ups; in Lightweight mode, cluster related topics into 3–4 batched rounds (see Two Sizes). Don't invent answers when the user is vague — ask or flag `[ASSUMPTION]`.

2. **Cross-cutting only.** The design doc names choices that affect many features or services. If an answer starts describing one specific feature's endpoints, event payloads, schema columns, or screens, stop and redirect: that's SPEC territory.

3. **Rationale, not just choice.** "Postgres" is not a design decision. "Postgres because we need transactions and relational queries; team knows it" is.

4. **State what you're ruling out.** For each decision, note what's explicitly not chosen (and why, if the reason isn't obvious).

5. **Push back on "we'll figure it out as we go."** A decision deferred to the first SPEC is still a decision — just an implicit one. If the user wants to defer, the doc should say so explicitly and flag which feature will settle it.

6. **Flag unknowns with `[ASSUMPTION]`** rather than filling them in silently.

## Interview Topics (in order)

Cover one at a time. Some may already be answered by the PRD — confirm, don't re-ask.

### Base topics (always)

1. **Size check** — lightweight or full RFC?
2. **Constraints from the PRD** — re-read the PRD. Extract non-negotiables: compliance (HIPAA, SOC2, GDPR), platform targets (iOS/Android/web), scale envelope (peak RPS, concurrent users), integrations with legacy systems, timeline / budget limits. Every "why?" in later topics must trace back to one of these constraints, not to preference or default.
3. **Tech stack** — language, framework, runtime, database. One-line rationale per choice, each referencing a constraint from topic 2.
4. **Architecture pattern** — monolith, modular monolith, microservices, serverless, event-driven, client-only. Why this shape. *The answer here determines whether the Distributed topics below apply.*
5. **Shared data model** — only entities that appear in multiple features/services (e.g., `User`, `Account`). Names, relationships, **and which service or feature owns the source of truth for each entity**. Columns, types, and indexes are deferred to the SPEC of the owning feature — the design doc settles ownership, not schema.
6. **API conventions** — REST / GraphQL / RPC, response envelope, pagination, auth header, error format. Applies to every endpoint.
7. **Auth approach** — user-facing strategy (JWT vs sessions vs OAuth-provider), not token shapes or refresh timings.
8. **Deployment model** — where and how this runs; what's managed vs self-hosted.
9. **Non-functional targets** — latency / availability / scale envelope *if* they constrain the choices above.
10. **Open questions / deferred decisions** — unresolved items, who decides, when.

### Distributed topics (only if architecture is microservices, event-driven, or multi-service)

Skip this whole block for monoliths and client-only apps.

10. **Service topology** — which services exist, what each owns (entity-level, not column-level). One-line purpose per service.
11. **Sync comms conventions** — gRPC vs HTTP, retry/backoff policy, circuit-breaker strategy, timeout defaults. Applies to every inter-service call.
12. **Async messaging** — broker choice (Kafka / RabbitMQ / SQS / Pub-Sub); topic/queue naming scheme; schema format (Avro / Proto / JSON Schema); delivery guarantees (at-most-once / at-least-once / exactly-once) + idempotency requirements; ordering rules. **Not specific event payloads** — those go in the SPEC of the feature that emits them.
13. **Data ownership & consistency** — which service owns which entity; where consistency is strong vs eventual; saga / compensation pattern if used.
14. **Inter-service auth** — mTLS, service tokens, or shared secret. How services identify each other.
15. **Observability conventions** — trace ID propagation, log format, required metrics per service. Applies to every service.

If any Distributed topic surfaces a concern that's really per-feature (e.g., *"what's in the book-added event"*), park it under `Notes for feature SPECs` and keep going.

### For full RFCs (any architecture)

Add: alternatives considered (per decision) and risks/mitigations.

## Output Format

Produce a design doc with these sections, in this order:

- Context (one paragraph — what problem, references the PRD)
- Tech Stack (with rationale)
- Architecture Pattern (with rationale)
- Shared Data Model (entities + relationships only)
- API Conventions
- Auth Approach
- Deployment Model
- Non-Functional Targets (if applicable)
- **Distributed sections (only if architecture warrants)**: Service Topology · Sync Comms Conventions · Async Messaging · Data Ownership & Consistency · Inter-Service Auth · Observability Conventions
- Open Questions / Deferred Decisions
- Alternatives Considered (full RFC only)
- Risks & Mitigations (full RFC only)
- **Requirements for Feature SPECs** (the contract — see section below)

**Must NOT include:** per-feature endpoint request/response bodies, column-level DB schemas (types, indexes, FK constraints), screen-by-screen flows, specific event payloads, specific service-to-service call signatures, rate limits, validation rules, specific error code catalogs, or any single-feature implementation detail. If the user volunteers such details, capture them **immediately** in a separate section titled `Notes for feature SPECs` — don't wait until the draft phase, park them the moment they surface.

**Where to save:** `docs/design.md` unless the project already uses a different convention. Confirm with the user if the structure is ambiguous.

## Requirements for Feature SPECs — the contract

Every design doc ends with a **Requirements for Feature SPECs** section. This is the contract every downstream SPEC must satisfy — populated from the conventions and constraints established in the interview. This section is what turns the design doc from a narrative into a governance artifact: a future SPEC author (human or AI) opens the design doc, reads this section, and knows exactly which rules their SPEC must honor.

Examples of well-formed entries:

- "Every endpoint MUST use the pagination envelope defined in §API Conventions."
- "Every service MUST propagate `traceparent` per §Observability Conventions."
- "Every SPEC touching the `User` entity MUST reference AuthService as the owner (§Service Topology)."
- "Every write path MUST define an idempotency key per §Async Messaging."
- "Every SPEC MUST state its worst-case latency against the NFR budget in §Non-Functional Targets."
- "Every SPEC handling PII MUST follow the retention policy in §Constraints (HIPAA)."

**Rules for writing this section:**

- Each bullet must be checkable (a reviewer can answer yes/no for any SPEC).
- Each bullet must reference the section of the design doc it derives from.
- Keep it short and enforceable. 5–10 bullets for lightweight; more permitted for full RFC.
- If a rule isn't worth enforcing on every SPEC, it doesn't belong here — leave it as prose in the relevant section.

## Kickoff Script

When the skill is invoked, open with:

> I'll help you write a design doc — the cross-cutting technical decisions that every feature SPEC will reference. I'll interview you one topic at a time, then draft.
>
> Design doc covers stack, architecture, shared data model (entities only), API conventions, auth approach, and deployment. If the architecture involves multiple services or async messaging, we'll also cover service topology, messaging conventions, data ownership, and observability. It does NOT cover per-feature endpoint shapes, column-level schemas, or specific event payloads — those live in each feature's SPEC.
>
> First: is this a lightweight architecture doc (solo / v1 / familiar choices) or a full RFC (contested / new service / cross-team)?

Then proceed through the topics in order, adding the Distributed block only if architecture answer warrants it.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Writing column-level DB schemas in the design doc | Entities + relationships only. Columns belong in the SPEC that uses the entity. |
| Specifying per-feature endpoint request/response bodies | Design doc sets conventions (pagination, error format). Endpoint shapes live in feature SPECs. |
| Specifying specific event payloads (e.g., fields on `book.added`) | Design doc sets messaging conventions (topic naming, schema format, delivery). Payloads live in the SPEC of the feature that emits the event. |
| Specifying token refresh timings, screen flows, validation rules | All SPEC territory. Design doc says "JWT-based auth"; SPEC says how long the token lives. |
| Describing a service's internal structure | The design doc names what a service owns and how it's reached. Its internals are that service's own docs/SPECs. |
| Inventing answers when the user is vague | Ask. If the user doesn't know yet, flag `[ASSUMPTION]` or add to Open Questions — don't silently guess. |
| Writing a full RFC for a solo weekend project | Offer the lightweight version first. A 3,000-word RFC for 200 lines of code is displacement activity. |
| Pushing back once, then writing exactly what the user asked for anyway | Pushing back is not compliance. Either wait for the user's explicit answer, or produce the correctly-sized doc — not the wrong one with a disclaimer. |

## Red Flags — STOP and redirect

If you catch yourself doing any of these, stop:

- Writing column types, index definitions, or FK constraints in the design doc
- Writing JSON request/response bodies
- Writing specific event payload fields
- Writing specific service-to-service call signatures
- Describing screen-by-screen flows or specific validation rules
- Producing 3,000+ words for a v1 or solo project
- Using words like "endpoint path," "status code 409," "token TTL," "index" in the output
- Filling in unknown cross-cutting decisions with your best guess instead of asking
- Stating a concrete choice in the doc (language, framework, broker, auth provider) that the user never confirmed, without tagging it `[ASSUMPTION]`
- Writing a feature SPEC because the user asked for one and "skipping" the design doc

All of these mean: pause, return to the cross-cutting question — or ask the user to clarify.

## Rationalizations — and their counters

| Excuse | Reality |
|---|---|
| "User wants the design doc comprehensive enough to code from" | You code from the SPEC, not the design doc. Design doc locks in cross-cutting choices so each SPEC doesn't reinvent them. |
| "User asked for the full DB schema in the design doc" | Entities + relationships go in the design doc. Column-level schema belongs in the feature SPEC that owns the entity. Capture columns under `Notes for feature SPECs`. |
| "User described the exact event payload — it's technical so it's design doc" | Messaging *conventions* (broker, schema format, delivery) are design doc. Specific payloads are SPEC. Park the payload under `Notes for feature SPECs`. |
| "User wants to skip the design doc phase — we're a small team" | A decision deferred to the first SPEC is still a decision, just implicit and inconsistent. A one-page design doc takes 10 minutes and saves weeks of drift. |
| "User asked for a rigorous RFC for a weekend project" | Ceremony should match stakes. Offer the lightweight alternative; if they still want the full RFC, confirm it's a learning exercise and deliver that framing. |
| "I already pushed back once — now I should respect their decision" | Pushing back once and then capitulating is the worst of both: the user gets the wrong artifact, and the disclaimer rots inside it. Offer the right artifact; if the user insists, produce what they asked for without a half-hearted objection buried in the doc. |
| "The user is vague, so I'll fill it in with reasonable defaults" | Silent defaults become binding contracts. Flag `[ASSUMPTION]` or ask — don't invent. |
| "This feature SPEC needs to exist before the design doc" | The SPEC will invent cross-cutting answers that the design doc is supposed to settle. Design doc first, SPEC after. |
| "The architecture is complex so the design doc must be deep" | Depth ≠ feature-level detail. A microservices design doc stays at topology, conventions, and ownership — it still doesn't list endpoint shapes or event fields. |

All of these mean: return to the cross-cutting question you were on, or explicitly confirm the size (lightweight vs RFC) with the user.

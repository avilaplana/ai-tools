# Full Conversation: Spec-Driven Development, Agile, and AI Coding Agents

A complete transcript of a conversation between a senior software engineer / tech lead and Claude, exploring how to adopt Spec-Driven Development (SDD) and AI coding agents in a multi-team brownfield context.

---

## User

I am a senior software engineer/Tech lead and I want to start using coding agents like Claude Code, OpenAI Codex or Github Copilot CLI to develop features.
I am really interested in apply Spec Driven Development as a methodology to apply the agile methodology.
Currently we have already built a platform that contains different components: UI component, synchronous components, asynchronous components. Some components have a HTTP inteface and others consume events from message broker. Some components store data in Warehouse and others in a remote cache, other communitcate downstream via HTTP and other publish messages to a message broker.
My team is not the owner of all the components. In fact we own part of the backend components.
I wonder how can it be the SDD process when you have a brownfield project across multiple teams. Can you help me to shape the problem ?

## Claude

Great question, and one worth unpacking carefully because most public SDD examples (spec-kit demos, blog posts) are greenfield single-team and quietly assume away exactly the constraints that dominate your reality. Let me try to reframe the problem in a way that surfaces the real decisions you'll have to make.

### What changes when SDD meets brownfield + multi-team

Greenfield SDD is mostly about *generating* a system from a spec. Brownfield multi-team SDD is mostly about *negotiating change* in a system that already exists, where most of the design decisions were made before you wrote your first prompt. That shift has three big consequences:

**The most valuable specs are not feature specs — they're contract specs.** In a single team, you can get away with feature-level specs because the team holds the system in its head. The moment ownership crosses team boundaries, the artifacts that matter most are the things crossing those boundaries: HTTP APIs, event schemas, message contracts, warehouse table contracts, cache key conventions. SDD only works smoothly when those are explicit, versioned, and owned. If they're currently tribal knowledge, your first SDD investment isn't a feature — it's writing down what already exists.

**The agent's biggest failure mode is missing context, not bad code.** Coding agents in greenfield are limited by their reasoning. In brownfield, they're limited by what they know about *your* world: existing patterns, conventions you take for granted, the reason that one weird abstraction exists, what's owned by whom, what you absolutely must not touch. The bottleneck moves from "can the agent code?" to "does the agent know enough to code *here*?"

**Specs replace design docs and RFCs, not just code comments.** In a multi-team setting, the spec becomes the artifact you share with other teams, get sign-off on, and use to negotiate contract changes. It's the thing that survives the conversation. That changes what a "good spec" looks like — it has to be readable by humans on other teams, not just executable by an agent.

### A layered spec model that fits your situation

Rather than a single "spec," think in layers, each with a different audience and lifecycle:

1. **Constitution / repo-level guidance** (`AGENTS.md`, `CLAUDE.md`, or similar). Architectural principles, coding standards, tech stack, testing approach, security baselines, and crucially the boundaries: which directories, services, or contracts the agent must not modify, and pointers to where shared contracts live. This is where you encode "the way we do things here." It's per-repo and changes slowly.

2. **Component specs** (one per service you own). Purpose, bounded context, public contracts in/out, data ownership, SLOs, dependencies on other teams' components. This is the "what is this thing" document. It changes when the component's role changes — rarely.

3. **Cross-team contract specs** (live in a shared place, ideally not your repo). OpenAPI for HTTP, AsyncAPI or schema-registry definitions for events, DDL or dbt contracts for warehouse tables, key/TTL conventions for cache. These are the negotiated boundaries. Changes here require coordination with the other team — that coordination is human work, not agent work.

4. **Feature specs** (one per change). What behavior is changing, which contracts change (if any), what the cross-team impact is, the technical plan inside your component, the task breakdown. This is where most of the spec-kit-style flow (`/specify` → `/clarify` → `/plan` → `/tasks` → `/implement`) actually lives. Most are short-lived and can be discarded after merge — though I'd keep them as ADR-like artifacts.

The reason this layering matters: an agent working a feature needs the constitution + relevant component specs + relevant contract specs as *context*, and produces the feature spec + code as *output*. If you collapse the layers, you'll either drown the agent in irrelevant context or starve it of the constraints it needs.

### The process loop, adapted for your reality

Roughly, for any non-trivial change:

**Discover.** Before specifying, have the agent (or you, with the agent) inventory what's affected: which of your components, which contracts, which downstream consumers, which upstream producers. In brownfield this step is non-negotiable and is often where the most surprises emerge.

**Specify behavior, not implementation.** What should the system do differently after this change, observable from the outside? Resist the urge to prescribe how. Specify the contract deltas explicitly and separately — those are the high-stakes parts.

**Negotiate cross-team impact early — and humanly.** If your spec implies a change to a contract you don't own, that becomes a conversation, an RFC, a meeting — not a PR an agent opens against another team's repo. The spec is the artifact that makes that conversation precise. This is the step most likely to break naive SDD adoption: agents will happily generate plans that quietly assume changes you can't actually make.

**Plan within your scope.** Only after contracts are settled, plan the implementation inside the components you own. This is where the agent earns its keep.

**Implement and verify.** Agents execute. But the verification layer matters more than usual: contract tests (Pact-style for HTTP, schema compatibility checks for events), integration tests against staging, and explicit checks that you haven't drifted from contracts you don't own. These tests are themselves spec artifacts — they encode the boundary.

### Where SDD genuinely pays off in your context, and where it bites

It pays off because it forces explicit contracts where you probably had implicit ones; it produces artifacts other teams can review without reading your code; it gives newcomers a coherent on-ramp; and it makes agent output auditable, which matters more when ownership is split.

It bites when (a) you treat all specs as equal — feature specs are cheap, contract specs are expensive, don't conflate them; (b) you let agents propose contract changes unilaterally, which leads to "we built it, now please accept it" anti-patterns with neighbouring teams; (c) you over-specify implementation, producing pseudocode-as-spec that's just code with extra steps; (d) you under-invest in the constitution layer and the agent keeps reinventing patterns you already have.

### Where I'd push you to make decisions

A few questions whose answers will materially shape the right SDD process for you, more than any general framework can:

- Do contracts (HTTP, events, warehouse schemas) currently live as machine-readable artifacts somewhere, or only in code/heads? If the latter, that's probably your first SDD project: extracting them.
- What's the current mechanism for cross-team change coordination — RFCs, design docs, meetings, Slack? SDD specs need to plug into that, not replace it.
- How wide is the autonomy you want to give agents — single-file edits, full features, or just scaffolding? That determines how rich your constitution and component specs need to be.

Happy to go deeper on any of these — for instance, what a concrete `AGENTS.md` looks like for a heterogeneous backend like yours, or how to structure feature specs so they pass cleanly between humans and agents, or how to handle the contract-spec workflow when the contract is owned by another team. Which thread is most useful to pull on first?

---

## User

if you were in charged what would you do? what steps would you take?

## Claude

Fair — let me drop the "it depends" hedging and tell you what I'd actually do, in order, with rough timing. I'm assuming you have a small team (3–6 engineers), real product pressure (you can't stop shipping for a quarter to "do SDD properly"), and that you want this to stick rather than be a one-quarter experiment.

### The guiding principle

I would absolutely not start with "let's adopt SDD across the team." I'd start by *using agents seriously myself for two weeks* before writing any process. The reason is that the right SDD process is shaped by where the agents actually fail in your codebase, and you can't predict that from blog posts. Most SDD methodology written today is over-engineered for the actual failure modes you'll hit.

The whole rollout below is built around: get signal from real use → codify what works → expand. Not: design the perfect process → roll out → discover it doesn't fit.

### Week 1–2: Use the agents yourself, instrument the pain

Pick Claude Code (or whichever agent your org will pay for — don't bikeshed this) and use it for every non-trivial task you personally do for two weeks. Real tasks in your real codebase, not toy projects. Keep a running note of every time the agent does something wrong, and *categorise* the failure: did it not know a convention? Did it modify something it shouldn't have? Did it invent an API that doesn't exist? Did it miss a cross-team contract? Did it produce code that passes tests but violates an unwritten rule?

This is the single most important step and the one most teams skip. The categories of failure you observe are exactly the headings of the documents you'll write next. If you skip this step you'll write a generic constitution full of platitudes that solves nothing.

Also during this period: notice what the agent does *well*. Where it just works, you don't need process — you need to get out of the way.

### Week 3: Write the constitution, painfully and specifically

Now, write `CLAUDE.md` (or `AGENTS.md`) at the root of each repo you own. Not a generic one. A specific one, where every rule maps to a failure you observed in weeks 1–2. Things like:

- "Database migrations live in `db/migrations` and use Flyway. Never modify a migration that has been merged to main; add a new one."
- "All HTTP handlers must go through the `RequestContext` middleware. Don't invent new error types — use the ones in `pkg/errors`."
- "This service publishes to the `orders.v2` topic. The schema lives in `<schema-registry-url>`. Do not change this schema without consulting the Payments team — flag any task that requires it."
- "Don't touch `legacy/` without explicit approval. Yes, even if it looks broken."
- Pointers to where the contracts you consume live, and which team owns each.

Keep it under 200 lines. If it's longer, the agent won't reliably attend to all of it and humans won't read it. Cut ruthlessly. Concrete > comprehensive.

This document is the highest-leverage artifact in your entire SDD adoption. Spend real time on it.

### Week 4: Extract one set of contracts that matter

Pick the *one* cross-team boundary that causes you the most pain — probably the HTTP API or event schema where you most often have miscommunication with another team — and make its contract a first-class machine-readable artifact. OpenAPI, AsyncAPI, JSON Schema, dbt contract, whatever fits.

Just one. Resist the urge to do them all. You're proving the pattern, not boiling the ocean.

Get the other team to at least acknowledge it as the source of truth, even if they don't formally adopt the same practice. Put a link to it in your constitution. From now on, agent-generated changes to anything that touches this boundary must reference this contract.

### Week 5–6: Run two real features through a lightweight spec flow, with you driving

Pick two upcoming features of medium size — not trivial, not huge. For each one, you personally drive an SDD-style flow:

1. Write a short feature spec (one page, max). What behaviour changes, what contracts are affected, what's in scope, what's explicitly out of scope, which other teams need to be looped in.
2. If contracts change, do that human conversation *before* generating any code. The spec is your artifact for that conversation.
3. Have the agent produce a plan from the spec. Read it critically. Push back. Iterate on the plan, not the code.
4. Have the agent implement. Review like you'd review a junior engineer's PR — but harder, because the failure modes are different (confident wrongness, subtle convention drift).
5. Afterwards, write a short retro: what worked, what didn't, what would you change about the spec template.

After two features you'll have a feature spec template that's specific to your context. Not borrowed from spec-kit, not borrowed from a blog post. Yours.

### Week 7–8: Bring in one other engineer, then the team

Pick the engineer on your team most likely to be a thoughtful critic — not the most enthusiastic adopter. Have them run the same flow on their next feature, using your constitution and template. Pair with them on the first one. Their friction points will reveal what's actually unclear in your artifacts vs. what was just obvious-to-you.

Update the constitution and template based on their feedback. Then roll to the rest of the team, with the expectation that the artifacts will keep evolving for another month.

### Month 3: Address the cross-team layer deliberately

By now you have working SDD inside your team. The cross-team layer is the next frontier and the harder one. I'd approach it like this:

- For each contract you own that other teams consume, publish it as a machine-readable artifact in a place those teams can reference. Treat changes to it as requiring a written change proposal — which is itself a small spec.
- For each contract you consume but don't own, write down (in your constitution or component spec) your current understanding of it and your assumptions. Share that with the owning team. This is often when you discover you've been wrong about something for two years.
- Don't try to make other teams adopt SDD. That's not your call and it'll create friction. Just make your interfaces with them clean, explicit, and reviewable. If your work is visibly higher-quality, interest from neighbouring teams often follows organically. If it doesn't, that's also fine.

### What I would explicitly *not* do

I wouldn't adopt a heavyweight framework like spec-kit on day one. Use it as inspiration, not as a starting point. Most of its structure is overkill until you've felt the pain it solves.

I wouldn't try to write specs for the whole existing system upfront. Specs for existing components emerge as a byproduct of working on those components. Writing them speculatively is busywork that goes stale.

I wouldn't let agents open PRs against other teams' repositories, even if technically possible. Cross-team changes go through humans. Always.

I wouldn't measure success by "number of features built with SDD" or "lines generated by agents." I'd measure it by: are reviews getting easier, are cross-team contract bugs decreasing, is onboarding faster, are engineers reporting that they trust the agent's output more over time. Those are the signals that this is actually working rather than just being performed.

I wouldn't build elaborate tooling early. No custom scripts, no orchestration, no spec-validation pipelines. Markdown files and discipline. Tooling comes after the practice is stable, not before.

### The honest summary

If I were in your seat, the single biggest mindset shift I'd try to internalise is: the spec is not a deliverable to the agent, it's a deliverable to your future self, your teammates, and other teams. The agent is incidentally also a consumer. Optimise for the humans and the agent benefits for free; optimise for the agent and you'll produce specs that no human wants to read or maintain, which is how this whole methodology dies in most teams within six months.

Start small, start with yourself, let the artifacts be shaped by real failures, and expand only when each layer is genuinely working. The teams that succeed at this don't have better frameworks — they have tighter feedback loops between practice and process.

Want me to sketch what a concrete constitution looks like for one of your component types — say a backend service that consumes events and exposes HTTP — so you have a starting shape to react to rather than a blank page?

---

## User

can you enumerate the list of artefacts that your process would generate with single line of explanation?

## Claude

Here's the full set, grouped by layer and roughly in the order they appear:

**Repo / team-level (written once, evolves slowly)**

1. `CLAUDE.md` (or `AGENTS.md`) — per-repo constitution encoding conventions, boundaries, and pointers to contracts; the highest-leverage artifact.
2. Component spec — one per service you own, describing its purpose, bounded context, contracts in/out, data ownership, and dependencies.
3. Failure log (weeks 1–2) — your running notes of where agents went wrong, used to derive the constitution; can be discarded after.

**Cross-team contract layer (shared, versioned, negotiated)**

4. HTTP API contracts — OpenAPI specs for endpoints you own or consume, treated as source of truth.
5. Event schemas — AsyncAPI or schema-registry definitions for topics you publish or subscribe to.
6. Warehouse / data contracts — DDL, dbt contracts, or equivalent for tables other teams read from.
7. Cache conventions — documented key formats, TTLs, and invalidation rules where shared with others.
8. Contract change proposals — short written proposals when a contract you own needs to change, used to drive the cross-team conversation.
9. Consumed-contract assumptions doc — your written understanding of contracts you depend on but don't own, shared with the owning team for confirmation.

**Per-feature (short-lived but worth keeping as ADR-like history)**

10. Feature spec — one-pager per change describing behaviour delta, affected contracts, scope, out-of-scope, and cross-team impact.
11. Implementation plan — agent-generated plan derived from the spec, reviewed and iterated by humans before any code.
12. Task breakdown — the plan decomposed into agent-executable units.
13. Feature retro note — short post-merge reflection on what the spec/plan got right or wrong, feeding back into templates.

**Verification layer (code, but spec-shaped)**

14. Contract tests — Pact-style HTTP tests and schema-compatibility checks that encode the boundaries in executable form.
15. Integration tests against staging — verifying you haven't drifted from contracts you don't own.

**Process / meta**

16. Feature spec template — your team's own template, derived from running the first two features, not borrowed.
17. Constitution changelog — lightweight record of why rules in `CLAUDE.md` exist, so future-you doesn't delete them by accident.

If I had to rank by importance: 1, 4–5, 10, and 14 are the ones that genuinely change outcomes. The rest are supporting cast — useful, but don't let their absence block you from starting.

---

## User

what does feature spec describe?

## Claude

A feature spec describes **what changes about the system's observable behaviour, and what that change costs in terms of contracts, scope, and coordination** — nothing more. It is not a design doc, not a technical plan, and not a ticket description. Those are different artifacts with different audiences.

Concretely, a good feature spec answers these questions, in roughly this order:

**Why** — one or two sentences on the problem or opportunity. Not a business case, just enough context that someone reading it in six months understands the motivation. If you can't write this in two sentences, the feature isn't well-understood yet.

**Behaviour delta** — what the system does differently after this change, described from the outside. "When a user submits an order with more than 10 items, the system splits it into batches of 10 and emits one `OrderBatchCreated` event per batch." Observable, testable, no implementation language. This is the core of the spec; everything else is supporting context.

**Scope and non-scope** — what's explicitly in, and crucially what's explicitly out. The non-scope list is usually more valuable than the scope list because it's where you head off the agent (and your teammates) generalising the change beyond what you intend.

**Affected contracts** — which HTTP endpoints, event schemas, warehouse tables, or cache keys are touched, and whether each is being added, changed (with version delta), or just consumed. This is where cross-team impact becomes visible. If this section has entries for contracts you don't own, the feature can't proceed until those conversations happen.

**Cross-team impact** — which other teams need to know, sign off, or change something. Often a direct consequence of the contracts section, but worth calling out separately because sometimes the impact is operational (on-call, dashboards, runbooks) rather than contractual.

**Acceptance criteria** — the conditions under which you'd call this done. Phrased as observable checks, ideally close to test cases. Three to seven bullets is usually right; if you have twenty, the feature is too big.

**Open questions** — things you don't know yet and need to resolve before or during implementation. Naming them explicitly is what separates a spec from wishful thinking. The agent should not be allowed to silently resolve these by guessing.

**Risks and rollback** — what could go wrong, and how you'd undo this if it did. For changes touching shared contracts or data, this section earns its keep; for purely internal changes it can be a single line.

A few things a feature spec deliberately does *not* contain:

It doesn't contain the implementation plan. That's the next artifact, generated from the spec, and kept separate so the spec stays stable while the plan iterates.

It doesn't contain code or pseudocode. The moment you start writing code in the spec, you've collapsed two layers and made the spec harder to review by anyone who isn't already in the implementation mindset.

It doesn't contain detailed designs for internal modules, class structures, or database schemas — unless those are part of the contract surface other teams or services see. Internal design lives in the plan or the code review.

It doesn't restate the constitution or component spec. It assumes those as context and only notes deviations.

The size test I'd use: a feature spec should fit on one page, be readable in three minutes by an engineer on another team, and contain enough information that *they* could tell you whether the change affects them. If it's longer than that, either the feature is too big and should be split, or the spec is doing work that belongs in another artifact.

---

## User

can you show me an example?

## Claude

Here's a concrete one. I'll invent a plausible scenario that matches your platform shape — backend service, mix of HTTP and events, cross-team dependency — so the spec has something real to bite into.

**Scenario:** Your team owns the `orders` service. Today it accepts an order via HTTP, persists it, and publishes a single `OrderCreated` event. The Fulfilment team (different team) consumes that event and has told you their downstream warehouse system chokes on orders with more than 10 line items. You've agreed to split large orders into batches before publishing.

---

### Feature Spec: Batch large orders on publish

**Status:** Draft · **Owner:** [you] · **Last updated:** 2026-04-30
**Affected component:** `orders-service` (owned by Orders team)
**Stakeholders:** Fulfilment team (consumer), Data Platform team (warehouse consumer)

#### Why

Fulfilment's downstream warehouse integration fails on orders with more than 10 line items, currently causing ~15 manual interventions per week. Splitting at publish time is cheaper than changing their integration and unblocks them this quarter.

#### Behaviour delta

When an order is accepted via `POST /orders`:

- If the order has 10 or fewer line items, system behaviour is unchanged: one `OrderCreated` event is published, as today.
- If the order has more than 10 line items, the system splits the line items into contiguous batches of up to 10 (preserving original order), and publishes one `OrderBatchCreated` event per batch. No `OrderCreated` event is published in this case.
- The HTTP response to the client is unchanged in both cases: `201 Created` with the order ID.
- Each `OrderBatchCreated` event carries the original `order_id`, a `batch_index` (0-based), `batch_count`, and the line items for that batch.

#### Scope

- Splitting logic in `orders-service` at publish time.
- New `OrderBatchCreated` event schema (v1).
- Metrics and logs for batch counts.

#### Non-scope

- No change to the `POST /orders` request or response shape.
- No change to order persistence — orders are still stored as a single record.
- No retroactive batching of historical orders.
- No change to `OrderCreated` schema; it remains as-is for the small-order path.
- Fulfilment's consumer changes are tracked separately by their team.

#### Affected contracts

| Contract | Direction | Change |
|---|---|---|
| `POST /orders` (HTTP) | We own | None |
| `OrderCreated` event (v1) | We own, Fulfilment consumes | None |
| `OrderBatchCreated` event (v1) | We own, Fulfilment consumes | **New** — schema in `schemas/orders/order-batch-created-v1.json` |
| `orders` warehouse table | Data Platform owns | None (still one row per order) |

#### Cross-team impact

- **Fulfilment team:** must subscribe to `OrderBatchCreated` and handle the new event before this rolls out. Confirmed in writing on 2026-04-22; their work is tracked in FULF-431.
- **Data Platform team:** notified for awareness; no action required as the warehouse table is unchanged.
- **On-call:** runbook update needed for the new event type and its alert thresholds.

#### Acceptance criteria

1. An order with 10 line items publishes exactly one `OrderCreated` event, identical to today.
2. An order with 11 line items publishes zero `OrderCreated` events and two `OrderBatchCreated` events, with `batch_index` 0 and 1 and `batch_count` 2.
3. An order with 25 line items publishes three `OrderBatchCreated` events with batch sizes 10, 10, 5.
4. Line item order is preserved across batches when reassembled by `batch_index`.
5. If event publish fails partway through a batched order, the order is not marked as published and the operation is retried as a whole (no partial publishes visible to consumers).
6. `OrderBatchCreated` schema passes backward-compatibility check in the schema registry CI job.
7. Contract test in `orders-service` verifies both small-order and batched-order paths against the published schemas.

#### Open questions

- **Threshold value:** is 10 the right cutoff, or should it be configurable? *Resolution needed before implementation.* Current assumption: hardcoded constant, revisit if Fulfilment's limit changes.
- **Idempotency:** Fulfilment has not confirmed whether they deduplicate on `(order_id, batch_index)`. Need confirmation before rollout, otherwise a retry could double-process a batch.

#### Risks and rollback

- **Risk:** Fulfilment's consumer is not ready when we deploy → batched orders silently dropped on their side. **Mitigation:** feature flag `orders.batch_publish_enabled`, default off; enable only after Fulfilment confirms readiness in staging.
- **Rollback:** disable feature flag. All orders revert to single `OrderCreated` event path. Schema registry entry for `OrderBatchCreated` v1 stays (no harm in keeping it).

---

A few things worth pointing out about this example, since they illustrate the principles more clearly than the abstract description did:

The behaviour delta is written in terms an engineer on the Fulfilment team could verify against their own system, without reading any of your code. That's the test for whether it belongs in the spec rather than the plan.

The non-scope list is doing real work — it pre-empts the agent (or a teammate) deciding to "also" modify the warehouse schema or the `OrderCreated` event "while we're in there." Without it, you'd find that scope creep in the plan or, worse, in the PR.

The contracts table is what makes cross-team impact concrete. If the `OrderBatchCreated` row instead said "Data Platform owns, change required," this whole feature would stop being an implementation task and become a coordination task — and the spec would correctly surface that *before* anyone wrote code.

The open questions are named, not silently assumed. The threshold question is the kind of thing an agent would happily pick a value for and move on; calling it out forces a human decision.

Acceptance criteria 1–4 are basically test cases in prose. That's deliberate — they're what the contract tests and integration tests will encode, so writing them this way means the verification layer almost falls out of the spec for free.

The whole thing is around 400 words of actual content. A reader on another team can absorb it in two or three minutes and tell you whether they need to care. That's the bar.

---

## User

probably before jumping in SDD and AI coding agents probably the first thing i need to review is the agile process in a company. how would it be process and the interactions between actors

## Claude

That's a sharp instinct, and honestly it's the realisation that separates teams who get value out of coding agents from teams who just generate more code faster. SDD and agents amplify whatever process you have; if the process is fuzzy, you'll get fuzziness at higher throughput. Let me try to lay out what a healthy agile process looks like with the actors and interactions made explicit, and then point at where AI agents specifically stress-test it.

### The actors, and what they're actually accountable for

Job titles vary wildly between companies, so I'll describe these by *role* rather than title. One person can hold multiple roles; that's fine and common.

**Product Manager / Product Owner.** Owns the *why* and the *what* at the problem level. Responsible for understanding user and business needs, prioritising, and saying no. Not responsible for how things are built. The signal they're doing the job well: engineers know why they're building something and what success looks like, and roughly half of proposed features get cut or deferred.

**Engineering Manager / Tech Lead.** Owns delivery and technical health. Tech leads in particular own the *how* at the architectural level — they shape technical direction, mentor, and are accountable for the long-term health of the codebase. Engineering managers add the people layer: growth, performance, hiring, team dynamics. In smaller teams these collapse into one person; in larger orgs they split.

**Engineers.** Own the *how* at the implementation level. Responsible for technical decisions inside their scope, for raising risks, for reviewing each other's work, and for the quality of what they ship. In a healthy team they push back on the PM when scope is wrong and on the tech lead when the design is wrong.

**Designer / UX.** Owns the user-facing experience. Works closely with PM on the problem and with engineers on what's feasible. Often missing from backend-heavy teams, which is fine, but someone still needs to own user-facing decisions if your component has them.

**QA / Test Engineers.** Own the quality strategy. In modern setups this is usually embedded — engineers write their own tests — but someone still needs to own the *strategy*: what we test, at which layer, with what tools. If nobody owns this, it defaults to "whatever the most recent engineer felt like," which decays fast.

**Stakeholders / partner teams.** Not part of your team, but consume what you produce or produce what you consume. In your case: the teams that own the components you don't, plus internal users of your platform. They're not in your stand-ups but they need to be in your loop at the right moments.

**Customers / users.** The actual humans the product exists for. Easy to forget in backend-heavy work; worth keeping a line of sight to them even if it's indirect (via PM, support, telemetry).

### The cadence: what happens at what frequency

Healthy agile is not about ceremonies; it's about **feedback loops at multiple timescales**. Each timescale answers a different question.

**Daily — "are we unblocked?"** A short sync, 15 minutes maximum, where engineers surface blockers and coordination needs. The failure mode is letting it become a status report to the manager. The test of a good stand-up: would the conversation still happen if the manager weren't there? If no, it's theatre.

**Weekly-ish (or per-sprint, if you sprint) — "are we building the right things?"** This is where the team picks up new work, refines what's coming, and demos what's done. Two distinct activities are usefully separated:

- *Refinement / grooming.* PM, tech lead, and engineers look at upcoming work and shape it together. The PM brings a problem; engineers ask hard questions; the tech lead flags technical implications. Output: work items that are well-enough understood to be picked up. This is where most teams underinvest, and it's the single highest-leverage meeting in agile. Bad refinement guarantees bad delivery.
- *Planning.* Team commits to what they'll do next. Short, mechanical, mostly a consequence of refinement having gone well.
- *Review / demo.* Team shows what's done to PM and stakeholders. The point is feedback, not approval. If it's an approval ritual you've turned agile into waterfall with extra steps.

**Per-iteration (every 2–4 weeks) — "are we working well?"** A retrospective. Team-only, no managers from outside the team, psychologically safe. Output: one or two concrete changes to how the team works. The rule I'd enforce: if a retro produces a list of ten action items, none of them will happen. Pick one.

**Quarterly-ish — "are we going in the right direction?"** Roadmap-level conversation between PM, tech lead, EM, and stakeholders. What's the next quarter's bet? What technical debt is biting? What partner-team dependencies do we need to negotiate? In your multi-team brownfield context this is also when you negotiate cross-team contracts and shared roadmap items.

**Continuously — "is what we shipped actually working?"** Telemetry, error rates, user feedback, support tickets. This is the most underrated loop. Most teams plan and demo but don't close the loop on whether the things they shipped solved the problem. If you shipped something three weeks ago and don't know whether it's used or working, your loop is broken.

### The interactions, traced through a single piece of work

Concretely, here's how a feature ought to flow through the actors. I'll keep it generic so it maps onto Scrum, Kanban, Shape Up, or whatever flavour you use.

1. **Problem identified.** PM, supported by user research, telemetry, or stakeholder request, identifies a problem worth solving. They write it down at the *problem* level, not the solution level. ("Fulfilment is hitting manual intervention 15× per week" not "build a batching system.")

2. **Triage.** PM, tech lead, and EM look at the problem alongside other candidates and decide it's worth investing in this iteration or quarter. Most ideas die here. That's healthy.

3. **Refinement.** PM brings the problem to the team. Engineers and tech lead ask: what does success look like, what constraints matter, what does the user actually do, who else is affected. The output is a shared understanding, not a spec yet. Cross-team dependencies are surfaced here — if the work touches another team's contracts, that conversation starts now, not at implementation time.

4. **Shaping / specifying.** A small number of engineers (often the tech lead plus one) shape the solution at the level of behaviour and contracts. This is where your feature spec lives. The PM is consulted on trade-offs but doesn't drive the spec; engineers do, because the spec is half technical. Cross-team contract changes are negotiated here, in writing, with the affected teams.

5. **Planning.** Once shaped, the work is sized and committed to. If shaping reveals it's bigger than expected, it goes back to refinement. Don't commit to half-shaped work; that's how teams ship the wrong thing on time.

6. **Implementation.** Engineers build, review each other's work, raise issues as they find them. The tech lead is available for design questions; the PM is available for scope questions. If an engineer hits a question that requires either to be summoned for a meeting, that's a sign refinement was incomplete — note it for the retro.

7. **Verification.** Tests at the levels the team has agreed on (unit, integration, contract, end-to-end as appropriate). For multi-team work, contract tests against partner teams' interfaces.

8. **Release.** Behind a flag if the change is risky or coordinated with another team. The PM and tech lead jointly own the decision to flip the flag, informed by telemetry and partner readiness.

9. **Verify in production.** Telemetry confirms the change does what it was supposed to. If it doesn't, the team takes that as a signal, not an embarrassment.

10. **Close the loop.** PM communicates outcome to stakeholders. Team reflects on whether the prediction matched reality. This is the step that, when done consistently, makes a team's estimates and prioritisation get measurably better over time.

### The interactions you'll get wrong if you're not deliberate

A few specific failure modes that I've seen repeatedly, worth naming so you can guard against them:

*PM hands engineers a solution instead of a problem.* This collapses refinement into "build this." Engineers stop thinking critically about whether it's the right thing to build. The fix is process discipline: PMs bring problems and constraints, the team shapes solutions together.

*Tech lead becomes a bottleneck for all decisions.* Healthy teams push decisions down to whoever has the context, with the tech lead setting principles and reviewing rather than approving each call. If every PR waits on the tech lead, you have a hierarchy, not a team.

*Stakeholders are looped in too late.* The classic version: you ship something that breaks an assumption another team had, and now it's a fire. The fix is naming cross-team impact during refinement and shaping, not at release time. Your contract layer (from the SDD discussion) is exactly the artifact for this.

*No one owns the "did it work" loop.* Most often the PM, but only if the team agrees. If no one does, the team stops learning from what it ships and the prioritisation feedback loop is broken.

*Retros surface the same problems repeatedly with no change.* Means action items aren't being owned or aren't being scoped small enough to actually do. One concrete change per retro, owned by a named person, beats ten aspirational ones every time.

### Where AI coding agents stress-test all of this

Now, looping back to your original direction: agents don't change this process, but they expose its weaknesses ruthlessly. Specifically:

If your refinement is fuzzy, agents will happily generate plausible-looking code for the wrong thing, faster than humans could. Bad refinement used to cost you a sprint; now it costs you a day, but the rework is the same.

If your cross-team contracts are tribal knowledge, agents will invent contracts that look reasonable and silently break partner teams. The pressure to make those explicit goes up sharply.

If your tech lead is a decision bottleneck, agents will route around them by making decisions implicitly in code — at scale. You'll find architectural drift you didn't sign off on.

If you don't close the loop on whether things worked, you'll generate even more code that nobody verifies the value of, and your throughput metrics will look great while your outcomes don't change.

So the order I'd actually suggest, given your instinct: spend a couple of weeks honestly auditing your current process against the picture above. Where are the loops broken? Where is ownership unclear? Where are stakeholders looped in too late? Fix the one or two worst ones first — and *then* introduce agents and SDD on top of a process that can absorb them. Doing it in the other order is how teams end up with a lot of generated code and no idea whether they're shipping the right things.

Want me to go deeper on any of the specific interactions — for instance, what good refinement actually looks like in practice, or how cross-team contract negotiation slots into the cadence — or sketch what an audit of your current process might look like in practice?

---

## User

i love this section: `The interactions, traced through a single piece of work` but i want to understand what artefacts, Epic, user stories, ADR, RFC, etc... are created by who and when. I would expect a map between `steps -> actors -> artefacts`

## Claude

Good — this is exactly the right thing to pin down, because most of the confusion in agile teams comes from fuzziness about who produces which artifact when. Let me map it out.

First, a quick taxonomy so the names mean the same thing, since these terms get used inconsistently across companies:

**Epic** — a large body of work spanning multiple features, usually a quarter or more of effort, owned by the PM. Answers "what user or business outcome are we pursuing." Not a giant feature; a thematic container.

**User story** — a single increment of user-visible value, sized to fit in an iteration. Owned by the PM in collaboration with the team. The "As a X, I want Y, so that Z" format is one convention; not sacred.

**Technical task / sub-task** — engineering work that supports a story but isn't independently meaningful to a user. Owned by engineers.

**Spike** — time-boxed exploration to answer an unknown before committing to a story. Owned by whoever has the question.

**RFC (Request for Comments)** — a written proposal for a non-trivial technical change, circulated for feedback before implementation. Owned by the engineer proposing the change. Used for things big enough that you want input but not necessarily a permanent decision record.

**ADR (Architecture Decision Record)** — a short, immutable record of a decision that was made, why, and what alternatives were considered. Owned by whoever drove the decision. ADRs are the *output* of decisions; RFCs are the *input*. An ADR can come out of an RFC, or out of a meeting, or out of a spike — the source doesn't matter, the durability does.

**Feature spec** — what we discussed earlier. Behaviour delta, contracts affected, scope. Sits between user stories and implementation.

**Implementation plan** — the technical decomposition of how to build a feature spec. Often agent-generated in an SDD flow.

The relationship roughly: Epic contains user stories. User stories may have feature specs (for non-trivial ones). Feature specs may reference RFCs and ADRs. Stories decompose into tasks. RFCs and ADRs are independent of the story hierarchy — they record decisions, which can span multiple stories or sit outside the story flow entirely.

### The map: steps → actors → artifacts

Now the actual mapping. I'll use the same ten steps as before. For each step I'll show the primary actor (who drives it), supporting actors (who participate), and the artifacts produced or updated.

**1. Problem identified**
- **Driver:** PM
- **Supporting:** users, telemetry, stakeholders, support team, engineers (informally)
- **Artifacts produced:** Problem brief or opportunity write-up — short, problem-level, not solution-level. Often lives as an item in the PM's backlog or a doc in the product area. May feed into or update an Epic if the problem is large enough.
- **Artifacts updated:** Roadmap, Epic (if the problem fits within an existing one).

**2. Triage**
- **Driver:** PM
- **Supporting:** tech lead, EM
- **Artifacts produced:** Prioritisation decision, sometimes captured as an ADR if it represents a meaningful product or technical bet ("we're investing in X over Y this quarter"). Usually captured more lightly as a roadmap update.
- **Artifacts updated:** Roadmap, Epic status, backlog ordering.

**3. Refinement**
- **Driver:** PM brings the problem; the team collectively refines.
- **Supporting:** tech lead, engineers, designer (if relevant), stakeholders from partner teams (if cross-team impact is suspected)
- **Artifacts produced:** User story (or stories) — written down, with acceptance criteria at the user-outcome level (not yet behaviour-delta level). If the work is large or unclear, a spike is created instead. If cross-team impact is identified, a flag is raised that an RFC or contract change proposal will be needed.
- **Artifacts updated:** Epic (stories added under it), backlog.

**4. Shaping / specifying**
- **Driver:** Tech lead or designated engineer
- **Supporting:** PM (on scope and trade-offs), engineers (on feasibility), partner teams (on contract changes)
- **Artifacts produced:**
   - *Feature spec* — behaviour delta, scope, affected contracts. The artifact we walked through in detail.
   - *RFC* — if the change involves a significant technical decision (new dependency, new pattern, new cross-team contract). Circulated for review.
   - *Contract change proposal* — if the change touches a contract owned by another team. Often a section within the RFC or a separate doc, depending on your conventions.
   - *Spike report* — if a spike was needed to resolve unknowns, the findings are written up here and feed the spec.
- **Artifacts updated:** User story (linked to the feature spec), Epic.

**5. Planning**
- **Driver:** Team (engineers commit to work)
- **Supporting:** PM (confirms priorities), tech lead (confirms readiness)
- **Artifacts produced:** Implementation plan or task breakdown — derived from the feature spec, decomposed into engineer- or agent-executable units. In an SDD flow this is often agent-generated and human-reviewed.
- **Artifacts updated:** Sprint/iteration backlog (if you sprint), story status moves to "ready" or "in progress."

**6. Implementation**
- **Driver:** Engineers (with agent assistance in your case)
- **Supporting:** Tech lead (design questions), PM (scope questions), reviewers
- **Artifacts produced:**
   - Code, obviously.
   - Tests at the agreed levels.
   - *ADR* — when a non-trivial decision is made during implementation that should outlive the feature ("we chose Postgres advisory locks over a Redis-based lock because…"). ADRs can be written mid-implementation; they don't have to come from an RFC.
   - PR descriptions linking back to the story and feature spec.
- **Artifacts updated:** Implementation plan (as reality diverges from plan), feature spec (if behaviour changes — but rarely; usually a behaviour change means going back to step 4).

**7. Verification**
- **Driver:** Engineers
- **Supporting:** QA (if a dedicated role exists), partner teams (for contract tests)
- **Artifacts produced:**
   - Contract tests (executable, lives in code).
   - Integration test results.
   - Test plan or test strategy doc — only if the feature warrants one; for most stories this is implicit in the acceptance criteria.
- **Artifacts updated:** Acceptance criteria checked off in the story.

**8. Release**
- **Driver:** Engineers (technical release), PM and tech lead jointly (decision to enable)
- **Supporting:** SRE/on-call, partner teams (if coordinated rollout)
- **Artifacts produced:**
   - Release notes / changelog entry.
   - Runbook update — operational doc for on-call, especially relevant for new event types or new failure modes.
   - Feature flag configuration.
- **Artifacts updated:** Story status moves to "released" or "deployed."

**9. Verify in production**
- **Driver:** PM (on outcome) and engineers (on technical health)
- **Supporting:** Telemetry, support, users
- **Artifacts produced:**
   - Outcome note — short write-up of whether the change achieved its intended outcome. Lives attached to the story or epic.
   - Incident report or postmortem — only if something went wrong.
- **Artifacts updated:** Epic status, story status moves to "done" only after this step (not at release).

**10. Close the loop**
- **Driver:** PM (for outcome communication), team (for retro)
- **Supporting:** stakeholders, EM (for retro)
- **Artifacts produced:**
   - Stakeholder update or demo summary.
   - *Retro notes* — what worked, what didn't, one or two action items.
   - *Feature retro note* — short post-merge reflection specifically on the feature spec / plan / agent flow, feeding back into your templates (this is the SDD-specific one from earlier).
- **Artifacts updated:** Process docs, templates, constitution (if a recurring failure mode is identified that should become a rule).

### The artifacts that live across steps, not within them

A few artifacts don't fit cleanly into a single step because they accumulate or evolve across the whole flow. Worth calling out separately:

**Epic** is created at the roadmap level (before step 1, really) and closed after step 9 when its constituent stories have all delivered their outcomes. It's updated continuously as stories under it are refined, shaped, and delivered.

**Roadmap** is updated at the quarterly cadence by PM and tech lead jointly, informed by everything happening in the iteration loops below it.

**Constitution / `CLAUDE.md`** is updated whenever a retro or a postmortem identifies a rule that should be encoded. It's not produced by any single step but evolves across them.

**Contract artifacts** (OpenAPI, AsyncAPI, schema registry entries) are updated in step 6 as part of implementation, but the *decision* to change them happens in step 4 via the RFC and contract change proposal.

**ADRs** can be written at almost any step from 4 onwards, whenever a decision is made that deserves to outlive the feature. The trigger isn't the step — it's the durability of the decision.

### Two patterns worth internalising

**The "who owns what" rule of thumb.** PM owns problem-level artifacts (epic, user story, outcome notes). Tech lead owns shaping-level artifacts (feature spec, RFC). Engineers own implementation-level artifacts (plan, code, tests, ADRs). Team owns process-level artifacts (retro notes, constitution updates). When ownership is ambiguous, default to "whoever is closest to the decision."

**The "input vs output" distinction.** RFCs, feature specs, and implementation plans are *inputs* to work — they shape what's about to happen. ADRs, outcome notes, and retro notes are *outputs* — they capture what did happen and why. Conflating these is a common smell. A feature spec is not an ADR; an ADR is not a retro note. Each has a different audience and a different lifespan.

If you wanted to make this concrete for your team, I'd suggest taking your next real piece of upcoming work and walking it through this map literally — at each step, ask "what artifact does this produce, who owns it, where does it live." You'll discover quickly which artifacts your team already produces (probably stories and PR descriptions), which it produces inconsistently (probably feature specs and ADRs), and which it doesn't produce at all (probably outcome notes — almost no team does this well). That gap analysis is more useful than any generic template I could give you.

Want me to do a worked example of the full flow for a single story — say, the order-batching feature from earlier — showing each artifact concretely at each step?

---

## User

could we say that a summary could be problem -> epic -> user stories -> tasks -> implementation ->.....
what is the feature spec? can you give me an example?

## Claude

Yes, that summary is roughly right, and it's a useful spine to hang everything else off. Let me complete it and then place the feature spec precisely within it.

### The full spine

**Problem → Epic → User stories → Feature spec → Implementation plan → Tasks → Code → Verification → Release → Outcome**

Reading left to right, each artifact answers a more specific question than the one before it:

- **Problem** — what's wrong or what opportunity exists in the world. Owned by PM.
- **Epic** — the thematic container for solving a class of problems over a quarter or so. Owned by PM.
- **User story** — one increment of user-visible value within the epic. Owned by PM, refined with the team.
- **Feature spec** — for non-trivial stories, the precise behaviour and contract change the system will undergo. Owned by tech lead or designated engineer.
- **Implementation plan** — how we'll build what the feature spec describes, inside our codebase. Owned by engineers (often agent-generated, human-reviewed).
- **Tasks** — the plan decomposed into executable units. Owned by engineers.
- **Code** — the actual change. Owned by engineers.
- **Verification** — tests and checks confirming the code does what the spec said. Owned by engineers.
- **Release** — the change reaches production, possibly behind a flag. Joint decision.
- **Outcome** — did this actually solve the problem we started with. Owned by PM, informed by telemetry.

### Where the feature spec sits and why it exists

The feature spec is the **bridge between the user story and the implementation plan**. It exists because user stories and implementation plans answer fundamentally different questions, and trying to go directly from one to the other is where most teams lose precision.

A user story says *"As an operations engineer, I want large orders to flow through fulfilment automatically, so that I don't have to manually intervene 15 times a week."* That's a problem statement framed around a user. It tells you what success looks like at the human level. It does *not* tell you what the system does differently, what contracts change, who else is affected, or what's out of scope.

An implementation plan says *"Add a `BatchSplitter` class to `orders-service`, modify the publisher in `OrderEventPublisher.kt` to invoke it, register the new schema in the schema registry, add three test cases."* That's a technical decomposition. It tells you what code to write. It does *not* tell you why, what the externally observable behaviour is, or whether the change is even compatible with what other teams expect.

The feature spec sits in between and answers the question neither of those answer: **"what does the system do differently after this change, and what does that change cost in terms of contracts, scope, and coordination?"** It's the artifact you can hand to the Fulfilment team to ask "does this match what you need?" without them having to read your code or understand your sprint planning. It's also the artifact an agent can use as the input to generate a sensible implementation plan.

To make the layering vivid, here's the same feature at each level of the spine, drastically shortened:

**Problem:** Fulfilment requires manual intervention ~15 times a week because their downstream system can't handle orders with more than 10 line items.

**Epic:** Reduce manual operational toil in the order-to-fulfilment pipeline this quarter.

**User story:** As an operations engineer, I want large orders to flow through fulfilment automatically, so that I don't have to manually intervene 15 times a week.

**Feature spec:** When an order with more than 10 line items is accepted, the system splits it into batches of up to 10 and publishes one `OrderBatchCreated` event per batch instead of a single `OrderCreated` event. The HTTP response is unchanged. New event schema, no warehouse impact, Fulfilment must be ready to consume the new event type before flag-on. *(Plus the full spec content — scope, contracts table, acceptance criteria, open questions, risks.)*

**Implementation plan:** Add a `BatchSplitter` in the publish path of `orders-service`. Register `OrderBatchCreated` v1 schema. Wire feature flag. Add contract tests for both event types. Add metrics. Update runbook.

**Tasks:** "Define schema." "Implement splitter." "Wire publisher." "Add tests." "Update runbook." Each a half-day to a day.

You can see the progressive narrowing: problem → outcome → behaviour → approach → steps. Each layer is *derived from* the one above it but adds information the layer above didn't have. And each layer has a different audience: the problem is for the PM and stakeholders, the story is for the team, the feature spec is for the team plus partner teams, the plan is for engineers and agents, the tasks are for whoever is doing the work.

### The example, in full

I gave you one earlier in the conversation — the order-batching feature spec — and that *is* the example of what a feature spec looks like at this point in the spine. If you want, scroll back to it; it shows what the artifact contains in detail.

The thing worth re-emphasising is what the feature spec is *not*:

It is not the user story. The story is shorter, framed around the user, and lives in your tracker (Jira, Linear, whatever). The feature spec is longer, framed around the system's behaviour, and typically lives as a markdown doc in your repo or wiki.

It is not the implementation plan. The plan describes code structure; the spec describes externally observable behaviour. You should be able to change the plan without changing the spec (different implementation, same behaviour). You should not be able to change the spec without revisiting the plan.

It is not the ADR. An ADR captures *a decision* — "we chose batching over consumer-side pagination because…" — and is immutable once written. The feature spec captures *a change* and is alive until the change is shipped, after which it can be archived.

It is not every story's responsibility. Many small stories — "add a missing column to the API response," "fix a bug in the validation logic" — don't need a feature spec. The acceptance criteria in the story plus the PR description are enough. Feature specs earn their cost when (a) the change has externally observable behaviour worth describing precisely, (b) contracts are involved, or (c) other teams need to understand the change without reading code. If none of those apply, skip it.

So the practical heuristic: every piece of work has a story; only meaningful pieces of work have a feature spec; every feature spec has an implementation plan; every plan has tasks. The spine is consistent, but not every artifact appears for every change.

---

## User

when i read acceptace criterias as Given When Then where is this fit?

## Claude

Good catch — Given/When/Then is a real and useful thing, and the question of where it fits is one of those small details that, once clear, makes the whole spine click into place.

### What Given/When/Then actually is

Given/When/Then (often called Gherkin syntax, from the Cucumber tool that popularised it) is a structured way of writing acceptance criteria as scenarios:

- **Given** — the precondition or starting state
- **When** — the action or event that triggers the behaviour
- **Then** — the observable outcome

It comes from Behaviour-Driven Development (BDD), where the explicit goal was to make acceptance criteria **executable** — the same text that humans read as a specification can be parsed by a tool and run as a test. That's the bit that's often forgotten: Given/When/Then wasn't invented to make stories prettier; it was invented so the spec and the test were the same artifact.

In practice, most teams use Given/When/Then *without* the executable part. They write scenarios in that format because it forces precision, even if no tool ever runs them. That's a perfectly legitimate use, just worth knowing it's the diluted version.

### Where it fits on the spine

Given/When/Then is a **format for acceptance criteria**, and acceptance criteria appear at two levels of the spine — at slightly different granularities:

**On the user story** — coarse-grained, user-outcome level. Acceptance criteria here describe what the user will be able to do or observe, framed at their level of abstraction. Often one to three scenarios per story.

**On the feature spec** — fine-grained, system-behaviour level. Acceptance criteria here describe exactly how the system behaves at its boundaries, often close to test cases. Usually three to seven scenarios.

The two are not duplicates; they're the same idea expressed at different altitudes. The story's criteria are what the PM and stakeholders sign off on. The spec's criteria are what the engineers and partner teams verify against. They should be consistent but not identical.

To make this concrete, going back to the order-batching example:

**At the user story level:**

> **Given** the operations team has been receiving alerts for large orders failing in fulfilment
> **When** a customer submits an order with more than 10 line items
> **Then** the order should flow through fulfilment without manual intervention

That's user-outcome language. It doesn't mention events, batch sizes, or schemas. The PM and the operations stakeholder can read it and confirm "yes, that's what we want."

**At the feature spec level:**

> **Given** the `orders-service` is healthy and the feature flag `orders.batch_publish_enabled` is on
> **When** an order with 25 line items is accepted via `POST /orders`
> **Then** zero `OrderCreated` events are published
> **And** three `OrderBatchCreated` events are published with `batch_index` 0, 1, 2 and `batch_count` 3
> **And** the batch sizes are 10, 10, 5 in order
> **And** the original line item ordering is preserved when reassembled by `batch_index`

That's system-behaviour language. Engineers can verify it; the Fulfilment team can verify it matches what they expect to consume; it can be turned almost mechanically into a contract test.

You can see the progression: the story's criteria say *what* in user terms, the spec's criteria say *what* in system terms. Neither says *how* — that's still the implementation plan's job.

### Where it does *not* fit

Given/When/Then is acceptance criteria. It's not a replacement for:

The **behaviour delta** section of a feature spec, which is prose describing the change holistically. Given/When/Then handles the specific scenarios; the behaviour delta gives the overall shape. Both are useful — the prose for understanding, the scenarios for verification.

The **implementation plan**, which is about code structure, not behaviour. Writing "Given a `BatchSplitter` exists, When `split()` is called, Then it returns batches" is a smell — that's a unit test, not an acceptance criterion. Acceptance criteria describe externally observable behaviour at the system or feature boundary, not the behaviour of internal classes.

The **contract specs** themselves. OpenAPI and AsyncAPI describe shape, not behaviour. Given/When/Then describes behaviour using those shapes. They're complementary.

### A few practical notes from teams that do this well

The format is most valuable when you actually let it discipline your thinking, which means resisting the temptation to write vague Givens and Thens. "Given a user, When they do something, Then it works" is technically the format and adds zero value. The format earns its keep when the Given is specific enough that you could set up that state, and the Then is specific enough that you could observe whether it happened.

Don't write Given/When/Then for criteria that aren't really about behaviour. "The code must be reviewed by two engineers" is a process criterion, not an acceptance criterion. "Performance must be under 100ms p99" is a non-functional requirement and is fine in Given/When/Then form but often clearer as a flat statement. Use the format where it pays for itself.

The number of scenarios is itself a signal. If a single feature spec has thirty Given/When/Then scenarios, the feature is too big and should be split. If a story has zero, either the story is trivial (fine) or the team is hand-waving (less fine). Three to seven per spec is the comfortable range I've seen.

Where teams using BDD tools (Cucumber, SpecFlow, behave) get the most value is when the Given/When/Then in the spec is *literally the same text* that runs as the integration test. That closes the loop — the spec doesn't drift from the test because they're the same file. If you're not using a BDD tool, you don't get that benefit, but the discipline of writing criteria in this format still produces clearer specs and clearer tests, even if they're maintained separately.

So in short: Given/When/Then is the format your acceptance criteria can take, on both the user story and the feature spec, and it fits inside those artifacts rather than being its own layer of the spine. If your team finds the format helpful, use it consistently at both levels — coarser at the story, finer at the spec — and the verification step almost writes itself.

---

## User

after all these agile concepts. I have heard that when people is using AI coding agents to define workflows similar to SDD what they do is after a conversation with the agent a PRD or Product Requirements Document is generated. once this is generated, there is a process to define specs where github issues are created for example and after that it comes the implementation phase comes. But all these phases does not match one to one with all these agile concepts that you have given me

## Claude

You're right that they don't match one-to-one, and the mismatch is genuinely confusing because the AI-coding-agent community has converged on its own vocabulary that overlaps with agile terms but doesn't mean the same things. Let me untangle it honestly, because the honest answer is more useful than pretending the two worlds map cleanly.

### What the AI-agent SDD flow typically looks like

The flow you're describing — and you'll see variations of it in spec-kit, in Kiro, in various blog posts and YouTube tutorials — usually goes something like:

1. **Conversation with the agent** to explore the idea.
2. **PRD generated** — a "Product Requirements Document" capturing what to build.
3. **Spec / design phase** — sometimes a separate technical design doc, sometimes just a more detailed elaboration.
4. **Task breakdown** — often as GitHub issues, or as a checklist file.
5. **Implementation** — the agent works through the tasks.

This flow exists because it's a clean, linear pipeline that maps well onto what an agent can do in a single project, often with a single person driving. It's been popularised heavily because it makes for great demos: one prompt to PRD, one command to tasks, one command to implementation, ship.

### Why it doesn't match agile cleanly

A few honest observations about the mismatch:

**The AI-agent flow is essentially single-team, single-feature, often single-person.** Most demos are someone building a side project or a self-contained feature alone. Agile, by contrast, is a *team* methodology designed to coordinate multiple people across multiple ongoing pieces of work, with stakeholders and dependencies. The AI flow doesn't have a place for "we're going to refine this with the team next Tuesday" because there is no team.

**"PRD" is being used loosely.** In traditional product management, a PRD is a heavyweight document — often 10+ pages, owned by a PM, covering market context, user research, success metrics, requirements, and constraints. It's the artifact that justifies investment in a piece of work. In the AI-agent flow, what people call a "PRD" is usually much closer to what we've been calling a feature spec: behaviour, scope, acceptance criteria. The label was borrowed from product management but the artifact is doing a different job. This is the single biggest source of confusion.

**The flow collapses several agile steps into one document.** What the AI-agent "PRD" typically contains corresponds to a mix of: parts of a user story (the why and the user outcome), most of a feature spec (behaviour delta, scope, acceptance criteria), and sometimes parts of a problem brief. Agile separates these because different actors own them and they evolve at different speeds. The AI flow merges them because, in a single-person/single-feature context, that separation doesn't pay for itself.

**"Spec" in the AI-agent flow often means design doc.** When someone says "after the PRD I generate a spec," they often mean what we'd call an *implementation plan* or technical design — how the system will be built. This is a different sense of "spec" than how we've been using it. SDD's "S" technically refers to specifications of behaviour, but in practice many of the popular flows use the word for technical design.

**GitHub issues as the task layer is just tooling.** This is fine and works well — issues are a perfectly reasonable place for tasks. But it doesn't change the underlying structure; it's the same as having tasks in Jira, Linear, or a checklist. The interesting question isn't where tasks live but who decomposes them and who reviews them.

So the AI-agent flow is best understood as a **simplified pipeline that compresses the agile spine for solo or near-solo work**. It's not wrong — for the context it's built for, it's effective. It just doesn't survive contact with multi-team, brownfield reality without modification.

### A rough mapping, with the honest caveats

If you held a gun to my head and asked me to map the typical AI-agent flow onto the agile spine, it'd look something like:

| AI-agent flow stage | Roughly corresponds to | What's missing |
|---|---|---|
| Conversation with the agent | Refinement | PM, team discussion, stakeholder input, spike exploration |
| PRD generated | User story + feature spec, fused | Epic context, problem framing at the business level, prioritisation |
| Spec / design phase | Implementation plan, sometimes RFC | Cross-team negotiation, ADRs as durable decisions, partner-team review |
| GitHub issues | Tasks | Team commitment, capacity planning |
| Implementation | Implementation | Code review by humans who hold context the agent doesn't, contract verification |
| (usually missing) | Verification beyond tests, release coordination, outcome verification | The whole right half of the spine |

The "what's missing" column is the important one. It's not that the AI-agent flow is wrong; it's that it covers the middle of the spine and quietly omits both ends. Problem framing is assumed (someone already knows what to build), and outcome verification is assumed (it works if the tests pass). For solo work on a side project, both assumptions are fine. For a team in your situation, both are where the real risk lives.

### How to think about this for your context

Given your reality — senior tech lead, multi-team, brownfield, real product pressure — here's how I'd reconcile the two worlds:

**Use the agile spine as the structural truth.** Problem → Epic → Story → Feature spec → Plan → Tasks → Code → Verification → Release → Outcome. That's the skeleton. Don't lose any of those layers, because each one is doing real work in a multi-team context.

**Use the AI-agent flow as a tactical pattern *within* the spine.** Specifically, the agent flow fits well from feature spec onward: once you have a feature spec, having the agent help generate the implementation plan, decompose it into tasks (GitHub issues or otherwise), and execute them is exactly where coding agents earn their keep.

**Don't let "PRD" be the entry point.** The AI-agent demos start with "let me generate a PRD." In your context, by the time you'd generate one, the team and PM have already been through problem framing, refinement, and shaping. The agent should be entering at step 4 or 5 of the spine, not at step 1. If you let the agent generate the equivalent of a PRD without that human work upstream, you'll end up with confidently-written documents for the wrong problem.

**Be ruthless about renaming.** If your team is going to use these tools, agree on what *you* call each artifact and stick to it. If you decide your team's word for the feature spec is "spec," fine — but then don't also call the implementation plan a "spec." The vocabulary confusion in this space is real and avoidable. Pick names, define them in your constitution, and use them consistently.

**Recognise what the AI-agent community has actually figured out.** A few things from that flow are genuinely good and worth borrowing: keeping artifacts as markdown files in the repo (close to the code), using the agent to draft the first version of a spec or plan from a conversation (saves time), and tight loops between spec → tasks → execution. These are real improvements over heavyweight document-driven processes. Don't dismiss the flow just because it's simplified.

**Recognise what it's missing for you.** The cross-team dimension is the big one. Nothing in the typical AI-agent flow handles "and this requires the Fulfilment team to change their consumer." The PRD-to-issues-to-implementation pipeline assumes everything is within reach. Your process has to add that layer back in, and that's where the agile concepts — refinement involving partner teams, RFCs for cross-team decisions, contract specs as shared artifacts — become non-negotiable.

### The honest summary

The AI-agent SDD flow and traditional agile are two different abstractions of the same underlying activity, optimised for different contexts. The agent flow is optimised for speed and solo execution; agile is optimised for coordination and outcome verification. Neither is wrong. The mistake is treating either as universally correct.

For your situation, the right move is to keep agile as the structural backbone, use the agent flow as an accelerator inside the implementation half of that backbone, and be deliberate about not letting the agent's tidy linear pipeline talk you out of the messy human steps that actually matter in a multi-team context — the refinement, the cross-team negotiation, the outcome verification. The agent makes the middle faster. It can't make the ends go away. Teams that pretend it can are the ones who end up shipping a lot of well-tested code that doesn't move any business metric.

If it'd help, I could sketch what a hybrid flow looks like concretely — showing where in your week-to-week agile cadence the agent enters, what it's allowed to draft, what stays human-only, and how the artifacts from the agile world feed into and out of the agent's working context. That's where the rubber actually meets the road.

---

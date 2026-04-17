# Spec-Driven Development: Primer

A guide to the artifacts, conventions, and workflow that connect an idea to working code using spec-driven development.

## 1. Core concepts

**PRD (Product Requirements Document)** — the *what* and *why*
- Audience: product, design, stakeholders, engineering
- Contents: problem, users, goals, success metrics, scope, user stories, constraints
- Deliberately avoids implementation detail

**Design Doc** — the *what approach* and *why this way*
- Audience: engineering (and tech leads / reviewers for full RFCs)
- Contents: tech stack, architecture pattern, shared data model, API conventions, auth approach, deployment model
- Settles cross-cutting decisions that span multiple features before individual SPECs are written

**SPEC (Technical Specification)** — the *how*
- Audience: engineers implementing + tests consuming
- Contents: data model, APIs, state transitions, edge cases, acceptance criteria
- Testable and precise; references Design Doc for cross-cutting decisions

**Tasks** — the *what to build*
- Audience: engineers and AI agents implementing
- Contents: implementation work items and test cases, derived from SPEC acceptance criteria
- Derived, not authored — each AC produces tasks and tests; tracked in repo alongside SPECs

**Relationship:** PRD → Design Doc → SPEC → Tasks → Code. Each artifact constrains the next. The pipeline flows downward; corrections flow upward (see Change protocol).

## 2. Features vs User Stories vs User Flows

```
Feature = what we offer           (e.g. "Book search")
 ├── User Story = what user wants  (e.g. "As a reader, I want to scan ISBN…")
 │   └── User Flow = how they get there (Scan → detect → preview → save)
```

- A **feature** contains many **user stories**
- A **user story** maps to one primary **flow** (plus error flows)

Note: this is the *structural* hierarchy, not the *discovery* order. In practice you discover user stories first (§3 step 4) and group them into features afterwards (§3 step 5) — the containment exists, you just build it bottom-up.

## 3. The artifact pipeline (idea → implementation)

```
IDEA
  ↓
1. Problem & Users           (who hurts, why)
2. Scenarios / JTBD          (when does pain happen)
3. Goals & Metrics           (what success looks like, numerically)
4. User Stories              ("As a X, I want Y so Z")
5. Features                  (capabilities)
6. Technical Decisions       (stack, architecture, data model, conventions)
7. User Flows                (step-by-step paths)
8. Acceptance Criteria       (testable conditions per story)
9. Requirements              (functional + non-functional)
```

- **PRD** covers steps 1–5 (+ high-level flows and success metrics)
- **Design Doc** covers step 6 — cross-cutting technical decisions settled before detailed flows begin
- **SPEC** covers steps 7–9 — precise flows, detailed acceptance criteria, full requirements

```
PRD (what/why)
  ↓
Design Doc (what approach, what trade-offs, what constraints)
  ↓
Feature SPECs (how, precisely, per feature)
```

## 4. PRD vs Design Doc vs SPEC — where does each section live?

| Section | PRD | Design Doc | SPEC |
|---|---|---|---|
| Problem statement | ✅ | | |
| Target users / personas | ✅ | | |
| Goals & non-goals | ✅ | | |
| Success metrics | ✅ | | |
| User stories | ✅ | | ✅ (as acceptance criteria) |
| High-level user flows | ✅ | | |
| Tech stack & rationale | | ✅ | |
| Architecture pattern | | ✅ | |
| Shared data model (entities, relationships) | | ✅ | |
| API conventions (format, auth, errors, pagination) | | ✅ | |
| Auth approach | | ✅ | |
| Deployment model | | ✅ | |
| Detailed flows with error states | | | ✅ |
| Feature list | ✅ | | |
| Feature-specific data model details | | | ✅ |
| API design / endpoints (per feature) | | | ✅ |
| State transitions | | | ✅ |
| Indexing / performance tuning | | | ✅ |
| NFRs (stated) | ✅ | | |
| NFRs (made testable) | | | ✅ |
| Error response formats | | ✅ (conventions) | ✅ (per feature) |

**Spec-drift** happens when SPEC content sneaks into the PRD. It's easy to spot in hindsight: look for data models, endpoint paths, status codes, JSON payloads, or database indexes in your PRD — those are SPEC content.

**Design-drift** happens when Design Doc content sneaks into either direction — cross-cutting decisions appearing in the PRD (too early) or being made ad-hoc inside individual SPECs (too late, risks inconsistency).

## 5. SPEC structure

### PRD → SPEC directly?

Rarely. When each SPEC is scoped to a single feature, there are **cross-cutting decisions** that no single feature SPEC owns:

- **Tech stack** — language, framework, database
- **Architecture pattern** — monolith, modular monolith, microservices
- **Shared data model** — entities that span features (e.g. `User`, `Book` appear in auth, library, search, shelves)
- **API conventions** — REST vs GraphQL, error format, pagination, auth headers
- **Auth approach** — JWT, sessions, OAuth provider
- **Deployment model** — where and how does this run

If you skip this step, either each SPEC reinvents these decisions (and they may conflict), or the first SPEC decides for everyone implicitly.

```
PRD → Design Doc → Feature SPECs → Tasks → Code
```

**Two levels of Design Doc:**

| Level | What it covers | When to use |
|---|---|---|
| **Lightweight architecture doc** | Cross-cutting decisions listed above, stated as choices with brief rationale | Almost always — even solo/v1 projects benefit |
| **Full RFC / Design Doc** | Alternatives analysis, risk matrix, trade-off discussion, reviewer sign-off | Contested approach, new service, cross-team impact |

For solo/v1 work, a lightweight architecture doc is usually enough — a few pages that nail down stack, data model shape, and conventions so that feature SPECs can reference them instead of debating them.

### One SPEC or many?

Modern default: **one SPEC per feature**.

```
specs/
├── auth.spec.md              # Google sign-in, JWT, token refresh
├── book-resolution.spec.md   # Scan + search + manual + dedup
├── library.spec.md           # CRUD on UserBook, status transitions
├── shelves.spec.md           # Shelves + ShelfBook
├── stats.spec.md             # Year/genre/goal aggregations
├── goals.spec.md             # Reading goals
├── connectivity.spec.md      # Offline behavior, fail-fast
└── shared/
    ├── data-model.md         # All entities
    └── api-conventions.md    # Pagination, error format, auth
```

| Approach | When | Trade-off |
|---|---|---|
| One big SPEC | Small product, solo dev | Easy to keep consistent; unwieldy past ~1,000 lines |
| One SPEC per feature | Most cases | Parallelizable; small blast radius per change |
| One SPEC per domain/service | Microservices, multiple teams | Matches team ownership |

### SPEC vs Design Doc

| | Design Doc / RFC | SPEC |
|---|---|---|
| **Purpose** | Debate & decide an approach | Define precise, buildable behavior |
| **Audience** | Engineers, tech leads, reviewers | Engineers implementing + tests |
| **Contains** | Problem, options, chosen approach, trade-offs, risks | Data model, APIs, acceptance criteria |
| **Tone** | Argumentative ("why this, not that") | Prescriptive ("exactly this") |
| **Lifespan** | Frozen snapshot | Living — kept in sync with code |
| **Testable?** | No | Yes |

**Design doc answers:** *"Why are we building it this way instead of another way?"*
**SPEC answers:** *"Exactly what should this thing do, down to inputs and outputs?"*

### Writing acceptance criteria — Given/When/Then

The recommended format for acceptance criteria in SPECs is **Given/When/Then**:

```markdown
## AC: User can add a book to their library

Scenario: successful add
  Given the user is on the library page
  When they submit a valid title and author
  Then the book is stored and appears in their library

Scenario: missing required fields
  Given the user is on the library page
  When they submit without a title
  Then a validation error is shown and no book is created
```

Why this format:
- **Each scenario maps directly to a test** — there's no ambiguity about what to test
- **Happy path and error paths are explicit** — separate scenarios, not mixed in a bullet list
- **It's behaviour, not implementation** — no mention of endpoints, components, or databases

### SPEC internal structure

A SPEC is more than acceptance criteria. ACs define the behaviour, but the SPEC also contains the technical details needed to implement it:

```markdown
# Library — SPEC

## Acceptance Criteria
Given/When/Then scenarios — behaviour, layer-agnostic.
(See "Writing acceptance criteria" above.)

## API Endpoints
POST /api/books
  Headers:  Authorization: Bearer <jwt>
  Body:     { title: string (required), author: string (required), isbn?: string }
  Response: 201 { id, title, author, isbn, createdAt }
  Errors:   400 { code: "VALIDATION_ERROR", message, details[] }
            409 { code: "DUPLICATE_ISBN", message }

## Data Model
Book: { id, title, author, isbn, status, createdAt, updatedAt }

## State Transitions
want-to-read → reading → read
(no backwards transitions allowed)

## Error Handling
(feature-specific error cases beyond API conventions)
```

Each section serves a different consumer:

| SPEC section | What it defines | Who consumes it |
|---|---|---|
| **Acceptance Criteria** | Behaviour (Given/When/Then) | Tests — scenarios become test cases |
| **API Endpoints** | Contracts (headers, body, responses, errors) | Backend tasks |
| **Data Model** | Entities, fields, types, constraints | Backend + frontend tasks |
| **State Transitions** | Valid states and allowed changes | Validation logic |
| **Error Handling** | Feature-specific error cases | Backend + frontend tasks |

The ACs say *"when they submit a valid title, the book is stored."* The API Endpoints section says *exactly what that request and response look like*. Tasks reference both — the AC tells you what to test, the technical sections tell you how to build it.

Note: cross-cutting conventions (auth headers, error format, pagination) live in the **Design Doc**, not repeated in every SPEC. The SPEC only defines what's specific to this feature.

## 6. Tasks — from SPEC to implementation

### What tasks are

Tasks are **derived from SPEC acceptance criteria**. They are not authored from scratch — each acceptance criterion in a SPEC produces one or more tasks. They bridge the gap between "what should this feature do" (SPEC) and "what did we build" (code + tests).

### Where tasks live

Task files live in the repo alongside their SPEC, mirroring the naming convention:

```
specs/
├── library.spec.md           (authored — what the feature does)
├── library.tasks.md          (derived — generated from SPEC ACs, tracked in repo)
├── shelves.spec.md
├── shelves.tasks.md
```

Why in the repo:
- **Git tracks the history** — what was planned, when it was completed, what changed
- **AI agents can read them** to know what to work on next
- **The traceability chain stays in one place** — SPEC → tasks → code

For teams that use project management tools (Jira, Linear, GitHub Issues), the task file is the **source of truth**. The PM tool mirrors it for assignment, boards, and workflow — not the other way around.

### Task naming convention

```
<FEATURE>-<number>: <imperative verb> <specific scope>
```

Examples:
- `LIBRARY-001: Create POST /api/books endpoint`
- `LIBRARY-002: Validate allowed status transitions`
- `SHELVES-001: Add shelf assignment to book detail view`

The prefix traces back to the SPEC (`LIBRARY` → `library.spec.md`). The description uses imperative mood — it completes the sentence *"This task will..."*

### From AC to tasks — the translation

Each Given/When/Then scenario in the SPEC produces two things in the task file:
1. **Tests** — translated almost verbatim from the scenario
2. **Tasks** — the implementation work needed to make those tests pass

SPEC (behaviour):
```markdown
## AC: User can add a book to their library

Scenario: successful add
  Given the user is on the library page
  When they submit a valid title and author
  Then the book is stored and appears in their library

Scenario: missing required fields
  Given the user is on the library page
  When they submit without a title
  Then a validation error is shown and no book is created
```

Task file (implementation):
```markdown
## AC: User can add a book to their library

### Tasks
- [ ] LIBRARY-001: Create POST /api/books endpoint
- [ ] LIBRARY-002: Validate required fields (title, author)
- [ ] LIBRARY-003: Store book in database, return created book
- [ ] LIBRARY-004: Build add-book form component
- [ ] LIBRARY-005: Call POST /api/books on submit

### Tests (from scenarios)
- [ ] **Test:** successful add — submit valid title and author → book is stored and appears in library
- [ ] **Test:** missing required fields — submit without title → validation error shown, no book created
```

The pattern: **scenarios become tests, tasks are the work to make them pass.** This is TDD at the spec level — you know what the tests are before you write a line of code.

- **Headings** are acceptance criteria, copied verbatim from the SPEC
- **Tasks** are implementation work items with IDs
- **Tests** are derived from Given/When/Then scenarios
- **Checkboxes** track status — git diffs show progress over time

### The traceability chain

```
SPEC acceptance criterion → Task ID → PR/commit → Test
```

This chain works in both directions. From any test, you can trace back to the SPEC requirement it verifies. From any AC, you can find the tasks, PRs, and tests that implement it.

### Features that span multiple layers

When a feature involves frontend and backend (or multiple services), the SPEC stays **layer-agnostic** — it defines behavior from the user's perspective. The layer split happens in the task file.

The SPEC is layer-agnostic — it uses Given/When/Then:

```markdown
## AC: User can add a book to their library

Scenario: successful add
  Given the user is on the library page
  When they submit a valid title and author
  Then the book is stored and appears in their library

Scenario: missing required fields
  Given the user is on the library page
  When they submit without a title
  Then a validation error is shown and no book is created
```

The task file breaks it down by layer:

```markdown
## AC: User can add a book to their library

### Backend
- [ ] LIBRARY-001: Create POST /api/books endpoint
- [ ] LIBRARY-002: Validate required fields (title, author)
- [ ] LIBRARY-003: Store book in database, return created book
- [ ] **Test:** POST with valid data returns 201
- [ ] **Test:** POST with missing title returns 400

### Frontend
- [ ] LIBRARY-004: Build add-book form component
- [ ] LIBRARY-005: Call POST /api/books on submit
- [ ] LIBRARY-006: Show validation errors from API response
- [ ] LIBRARY-007: Navigate to library view on success
- [ ] **Test:** form submits and book appears in library
- [ ] **Test:** missing fields show inline errors

### Dependencies
- LIBRARY-004 depends on LIBRARY-001 (API must exist before frontend calls it)
```

The principle: **SPECs describe behavior, tasks describe implementation.** Layers are an implementation concern.

| Artifact | Cares about layers? | Why |
|---|---|---|
| **SPEC** | No | Defines behavior from the user's perspective |
| **Design Doc** | Yes | Defines how layers communicate (API conventions, data model) |
| **Tasks** | Yes | Breaks behavior into implementable work per layer |

## 7. Good-practice notes

### Writing a PRD
- **Interview before drafting.** Don't write the PRD linearly top-to-bottom — brain-dump, group, then structure.
- **One topic at a time.** Dumping 20 questions at once produces shallow answers.
- **Push back on vagueness.** "Users want it to be fast" → "How fast? Measured how? Compared to what?"
- **Separate goals from non-goals explicitly.** For every goal, ask what's deliberately out of scope.
- **Flag assumptions.** Mark unverified claims `[ASSUMPTION]` so they can be validated later.

### Change protocol
When implementation reveals that a SPEC or Design Doc assumption is wrong, **stop coding**. Update the upstream document first, then continue. The pipeline flows downward (PRD → Design Doc → SPEC → Code), but corrections flow upward. The rule is simple: no code should be written against a known-wrong SPEC.

### The common mistakes
- **Spec-drift** — including data models, endpoints, schemas in the PRD
- **Solution-drift** — prescribing HOW instead of WHAT ("add a dropdown" vs "let users filter")
- **Goal-less features** — shipping something with no measurable success criteria
- **Everything-is-must-have** — no prioritization means everything is p0, which means nothing is

### From idea to PRD, practically
1. Brain-dump problem, users, scenarios in bullet form
2. Extract 3–5 goals with measurable success criteria
3. Write stories — one line each
4. Group stories into features
5. Sketch flows for the top 3 golden-path stories only
6. Add acceptance criteria for v1 stories
7. Requirements come last — you can't write good NFRs without seeing the product shape

## 8. Tooling

### `creating-prds` skill

Personal Claude Code skill at `~/.claude/skills/creating-prds/SKILL.md`.

Triggers when the user wants to write a PRD from scratch. Runs a structured interview covering: problem, users, goals, non-goals, user stories, flows, scope, constraints, open questions — then produces a PRD that explicitly excludes SPEC-level content (data models, APIs, etc.).

### `creating-design-docs` skill

Personal Claude Code skill at `~/.claude/skills/creating-design-docs/SKILL.md`.

Triggers when a PRD is complete and the user is about to start technical work. Runs a structured interview over cross-cutting decisions only: tech stack, architecture pattern, shared data model (entities + relationships), API conventions, auth approach, deployment. If architecture is microservices / event-driven, adds a Distributed block covering service topology, sync comms conventions, async messaging (conventions, not payloads), data ownership, inter-service auth, and observability. Offers two sizes — lightweight architecture doc or full RFC — and right-sizes ceremony to stakes.

**How to verify a skill was invoked:**
- Look for a `Skill(<name>)` tool call in the transcript
- Claude announces: *"Using <name> to..."*
- The skill's kickoff script appears at the start of the response

**Trust the tool call, not the narration.** If Claude claims to use the skill but no tool call appears, it's hallucinating the invocation.

## 9. Suggested workflow going forward

```
PRD.md                        (authored — product intent, steps 1–5)
  ↓
docs/design.md                (authored — cross-cutting tech decisions)
  ↓
specs/<feature>.spec.md       (authored — precise feature behavior)
  ↓
specs/<feature>.tasks.md      (derived — generated from SPEC ACs, tracked in repo)
  ↓                            ↘
code + tests                   PM tool (optional — assignment, boards, workflow)
```


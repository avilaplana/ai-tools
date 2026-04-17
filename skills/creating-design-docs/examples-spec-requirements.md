# Examples: Requirements for Feature SPECs

Three worked examples of the "Requirements for Feature SPECs" section, one per sizing/complexity profile. These stress-test the format — can it express what we need at different scales, is each bullet checkable, does every rule trace back to a design doc section.

---

## Example 1 — Lightweight (solo mobile app)

Context: "Shelf" reading tracker, solo dev, Expo + Supabase BaaS, single region, 4 base features (auth, add book, book list, status).

```markdown
## Requirements for Feature SPECs

- Every SPEC that reads or writes a record MUST use the Supabase client SDK — no direct Postgres connections (§Tech Stack).
- Every SPEC touching user identity MUST treat Supabase Auth as the owner; user profile data lives in `auth.users` metadata, not a separate `users` table (§Shared Data Model).
- Every SPEC MUST declare its offline behavior: `requires-connection` or `cached-until-sync` (§Constraints — mobile networking).
- Every SPEC that lists items MUST use cursor-based pagination with `limit ≤ 50` (§API Conventions).
- Every SPEC MUST state its target p95 latency; if > 200 ms, explain why (§Non-Functional Targets).
- Every SPEC MUST treat Supabase RLS rejections as `PERMISSION_DENIED` → redirect to sign-in (§Auth Approach).
```

6 rules. Matches the "5–10 for lightweight" guidance. Every rule is a yes/no check; every rule traces to a section.

---

## Example 2 — Microservices (marketplace)

Context: multi-team marketplace, services `Identity / Listings / Orders / Payments / Notifications`, Kafka async, Postgres per service, gRPC for sync, Auth0 OIDC + JWT between services.

```markdown
## Requirements for Feature SPECs

- Every SPEC MUST declare the owning service; cross-service dependencies MUST be listed with expected RPS and failure behavior (§Service Topology).
- Every SPEC that reads or writes an entity MUST call the owning service — no cross-database reads, ever (§Data Ownership & Consistency).
- Every sync endpoint in a SPEC MUST use gRPC + Proto; REST is permitted ONLY at the edge gateway (§Sync Comms Conventions).
- Every SPEC emitting an event MUST: (a) define the Avro schema under `schemas/<service>/<event>.avsc`; (b) use the topic name `<domain>.<entity>.<action>`; (c) document the partition key; (d) guarantee consumer idempotency (§Async Messaging).
- Every multi-service write MUST define a saga / compensation path with explicit failure states (§Data Consistency).
- Every SPEC MUST propagate `traceparent` on sync calls and a `trace-id` header on Kafka messages (§Observability Conventions).
- Every service-to-service call MUST include the service-token auth header; user JWTs MUST NOT be reused for S2S (§Inter-Service Auth).
- Every SPEC touching monetary amounts MUST use `amount_cents` (integer) + `currency` (ISO 4217). No floats (§Constraints — financial accuracy).
- Every SPEC MUST declare which endpoints are user-facing (JWT required) vs internal (service token) (§Auth Approach).
- Every SPEC MUST state retry/backoff policy; default is exponential backoff, max 3 attempts, circuit-breaker trips after 5 consecutive failures (§Sync Comms Conventions).
```

10 rules. At the upper end of RFC territory but every one is load-bearing for multi-team work.

---

## Example 3 — Compliance-heavy (HIPAA health records)

Context: Go + Postgres on AWS, modular monolith, Patient/Record/Provider entities owned by the `PHI` module, OAuth2 + SMART-on-FHIR, HIPAA + SOC2 compliance required.

```markdown
## Requirements for Feature SPECs

- Every SPEC that reads or writes PHI MUST enumerate the PHI fields accessed and justify minimum-necessary (§Constraints — HIPAA §164.502(b)).
- Every SPEC MUST emit an audit log entry on every PHI access: `{actor, action, resource, timestamp, justification}` (§Observability).
- Every SPEC MUST use the `@phi_access` middleware for PHI operations — no inline audit logging (§Tech Stack).
- Every SPEC persisting PHI MUST encrypt at rest (AES-256 via AWS KMS) and in transit (TLS 1.3) (§Constraints — HIPAA §164.312(a)(2)(iv)).
- Every SPEC exposing PHI MUST require SMART-on-FHIR scopes; missing scope returns 403, not 401 (§Auth Approach).
- Every SPEC MUST declare its data retention window; default is 7 years from last access; deviations require compliance review (§Constraints — HIPAA §164.530(j)).
- Every SPEC involving a third-party service MUST cite the signed BAA and effective date (§Constraints — HIPAA §164.308(b)).
- Every SPEC MUST support the break-glass access path: emergency read with mandatory justification, logged to a separate audit stream (§Constraints — clinical requirement).
- Every SPEC modifying Patient or Record MUST emit a `patient.record.modified` event for audit replay (§Async Messaging).
- Every SPEC MUST declare RPO / RTO; defaults: RPO = 15 min, RTO = 1 hr (§Non-Functional Targets).
- Every SPEC handling patient data-portability MUST support FHIR R4 export (§Constraints — HIPAA §164.524).
```

11 rules. Heavy, but every bullet is load-bearing for an audit. Note how regulatory clauses get cited directly in the traceability marker.

---

## Format observations

What the format handles well:
- **Standing rules** across all SPECs in the project ("Every SPEC MUST…")
- **Conditional rules** for categories of SPEC ("Every SPEC touching PHI MUST…")
- **Traceability** via the trailing `(§Section)` anchor
- **Scaling**: 6 rules at lightweight, 11 at compliance-heavy — still scannable

What the format does NOT handle:
- **Ordering constraints** ("the first SPEC to introduce X must…") — those belong in a migration plan or task list, not here.
- **Soft guidance** ("prefer Postgres for relational data") — belongs in the Tech Stack section as prose.
- **Per-SPEC exceptions** — this is a contract, not a negotiation.

## Tweaks to fold back into the skill

1. **Use RFC 2119 vocabulary** (MUST / SHOULD / MAY). Default is MUST; SHOULD is for rules with documented exceptions.
2. **Upper bound**: cap at 15 rules even for full RFC. Beyond that, the checklist is a checklist of checklists — split the design doc.
3. **Section anchor format**: every bullet ends with `(§<Section Name>)`. If a rule derives from an external citation (like HIPAA §164.502(b)), nest it: `(§Constraints — HIPAA §164.502(b))`.
4. **Checkability test**: before adding a bullet, ask "can a reviewer answer yes/no about a given SPEC?" If not, it's prose, not a rule.

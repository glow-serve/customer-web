# GlowServe

A multi-role, location-aware marketplace for booking verified beauty, grooming, spa and wellness
professionals at a customer's home. It coordinates service discovery, pricing, scheduling,
professional assignment, payment, OTP-controlled service execution, ratings, support, professional
earnings and marketplace administration.

**Core promise:** a customer can find a service available at their address, reserve a valid time
slot, complete a transparent checkout, receive a verified professional, track the service lifecycle,
and obtain support or a refund through a controlled, auditable workflow.

> **Status: pre-implementation.** This repository currently contains only the specification. Nothing
> below the "Architecture" section is running code yet — treat it as the build target, not a
> description of what exists. Source of truth for scope is `GlowServe_PRD_PERN_Marketplace_v1.0` (PRD v1.0, 30 Jul 2026).

---

## Product surfaces

| Surface | Primary users | Purpose |
| --- | --- | --- |
| Customer app | Customers | Discover, book, pay, track, reschedule, cancel, review, get support |
| Professional app | Service professionals | KYC, availability, accept work, execute services, track earnings |
| Admin / Operations app | Ops, support, finance, catalog, admins | Run the marketplace, resolve exceptions, manage catalog |
| Backend platform | All clients + integrations | Business rules, transactions, authorization, events, data integrity |

## Product principles

- Transparent pricing before payment — taxes, fees, discounts and refund conditions included.
- **Server-authoritative** booking and payment state; a client confirmation is never sufficient.
- Professional trust via KYC, skill mapping, OTP checkpoints and auditable status history.
- Operationally recoverable: unassigned, delayed, cancelled, disputed and failed-payment bookings
  stay actionable.
- Permission enforcement on the backend for every protected operation.
- Mobile-first customer and professional experiences; desktop-first administration.

---

## Tech stack

| Layer | Choice |
| --- | --- |
| Database | PostgreSQL (system of record) |
| API | Node.js + Express, TypeScript (strict) |
| Web | React + TypeScript |
| Cache / jobs | Redis + BullMQ — slot-hold expiry, assignment timers, notification retries |
| Real-time | Socket.IO with authenticated rooms |
| Files | Object storage (private buckets, signed URLs) for KYC, evidence, invoices |
| Delivery | Docker / Docker Compose, CI-enforced lint + tests, generated OpenAPI |

Architecture is a **modular monolith**. Services get extracted only when there is a measured
operational need — not before.

## Architecture

```
Customer React App   Professional React App   Admin / Ops App
        \                    |                     /
         \                   |                    /
          ---->  Node.js + Express API gateway  <----
                 (JWT, validation, rate limits, request tracing)
                                |
   auth & RBAC · catalog & pricing · booking & slots · assignment · payments
   professional ops · reviews & support · notifications · reporting & audit
                                |
   PostgreSQL        Redis + BullMQ       Object storage      External
   (txns, constraints) (cache, jobs)      (KYC, evidence)     (gateway, maps, email/SMS/push)
```

### Backend module boundaries

`auth · sessions · RBAC` — `customers, addresses, serviceability` — `catalog, pricing, promotions` —
`carts, availability, slot holds, checkout` — `bookings, transitions, rescheduling` —
`professionals, KYC, skills, availability` — `assignment + operational exceptions` —
`payments, refunds, invoices, reconciliation` — `wallet, commission, settlements` —
`reviews, support, notifications, audit`

---

## Booking state machine

```
DRAFT → SLOT_HELD → PAYMENT_PENDING → CONFIRMED → ASSIGNMENT_PENDING → ASSIGNED
      → PROFESSIONAL_EN_ROUTE → PROFESSIONAL_ARRIVED → IN_PROGRESS → COMPLETED → PAYMENT_SETTLED
```

`SLOT_HELD → CONFIRMED` directly when an enabled pay-after-service method passes its checks.

Exception outcomes: `PAYMENT_FAILED`, `ASSIGNMENT_FAILED`, `CANCELLED_BY_{CUSTOMER,PROFESSIONAL,ADMIN}`,
`CUSTOMER_NO_SHOW`, `PROFESSIONAL_NO_SHOW`, `REFUND_PENDING`, `PARTIALLY_REFUNDED`, `REFUNDED`,
`DISPUTED`.

**Enforcement rules**

- One domain service validates every transition. Controllers never write `status` directly.
- Each transition writes `booking_status_history` in the *same transaction* as the status update.
- Domain events publish through an outbox after commit — never inside the transaction.
- Idempotency keys on payment confirmation, completion and retryable admin actions.

---

## Data model

PostgreSQL is the system of record. Money is stored as integer minor units (or exact numeric) with
an explicit currency. Timestamps are UTC, converted only at presentation boundaries.

| Domain | Tables |
| --- | --- |
| Identity | `users`, `user_sessions`, `roles`, `permissions`, `role_permissions`, `user_roles` |
| Customer | `customer_profiles`, `customer_addresses`, `customer_credits` |
| Professional | `professional_profiles`, `professional_documents`, `professional_skills`, `professional_service_zones`, `professional_availability`, `professional_time_off` |
| Catalog | `cities`, `service_zones`, `categories`, `services`, `service_variants`, `service_addons`, `service_prices`, `service_price_versions` |
| Commerce | `carts`, `cart_items`, `slot_holds`, `bookings`, `booking_items`, `booking_status_history` |
| Assignment | `assignment_attempts`, `booking_assignments`, `professional_location_snapshots` |
| Execution | `booking_otps`, `service_checklists`, `booking_evidence`, `booking_incidents` |
| Payments | `payment_orders`, `payments`, `payment_events`, `refunds`, `invoices` |
| Professional finance | `professional_wallets`, `wallet_transactions`, `settlements`, `settlement_items` |
| Growth | `coupons`, `coupon_rules`, `coupon_redemptions`, `memberships`, `customer_memberships` |
| Quality & support | `reviews`, `review_dimensions`, `support_tickets`, `support_messages`, `support_actions` |
| Platform | `notifications`, `notification_preferences`, `audit_logs`, `outbox_events`, `system_config_versions` |

### Constraints that carry the correctness

| Object | Requirement |
| --- | --- |
| `users` | Unique normalized email and/or phone; status index |
| `user_sessions` | Unique token-family id; indexed by user and expiry |
| `service_prices` | Unique active price context per service/variant/add-on, location and effective window |
| `slot_holds` | Indexed by expiry and customer; expired holds cannot convert |
| `booking_assignments` | Partial unique index — one active accepted assignment per booking |
| professional bookings | Range-conflict strategy: time ranges + locking or exclusion constraints |
| `payment_events` | Unique `(gateway, gateway_event_id)` for webhook idempotency |
| `coupon_redemptions` | Unique constraints + transaction-safe counters for per-user and global limits |
| `reviews` | Unique `(booking_id, customer_id)` |
| `audit_logs` | Indexed by actor/action/entity/created_at; no update or delete via the app role |

---

## API surface

Base path `/api/v1`. JSON everywhere except multipart file uploads. Every error uses one envelope:
`code`, `message`, field errors, request id, safe metadata. Cursor pagination for feeds and
timelines; page pagination for admin tables that need counts. `Idempotency-Key` is required on
booking creation, payment confirmation, refund creation and payout requests. OpenAPI is generated
and validated in CI.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| POST | `/auth/register` `/auth/login` `/auth/refresh` `/auth/logout` | Identity and session lifecycle |
| GET | `/catalog/categories` `/catalog/services` `/services/{id}` | Browse and search catalog |
| GET | `/availability` | Bookable slots for a cart + address |
| POST | `/slot-holds` | Create an expiring hold |
| GET/POST/PATCH | `/cart` `/cart/items` `/cart/items/{id}` | Server-side cart |
| POST | `/checkout/quote` | Authoritative quote |
| POST | `/payments/orders` `/payments/webhooks/{gateway}` | Gateway order + verified events |
| POST/GET | `/bookings` `/bookings/{id}` `/bookings/{id}/cancel` `/bookings/{id}/reschedule` | Booking lifecycle |
| GET/POST | `/professional/requests` `/professional/requests/{id}/accept\|reject` | Assignment offers |
| POST | `/professional/bookings/{id}/status\|verify-start\|verify-completion` | Execution + OTP |
| POST | `/reviews` `/support/tickets` | Quality and support |
| GET/POST | `/admin/operations/queues` `/admin/bookings/{id}/assign` `/admin/reports/{report}` | Operations |

### Error codes

`AUTH_INVALID_CREDENTIALS` · `AUTH_SESSION_REVOKED` · `PERMISSION_DENIED` ·
`LOCATION_NOT_SERVICEABLE` · `SLOT_UNAVAILABLE` · `PRICE_CHANGED` · `INVALID_BOOKING_TRANSITION` ·
`PAYMENT_PENDING_VERIFICATION` · `PAYMENT_EVENT_INVALID` · `REFUND_LIMIT_EXCEEDED` ·
`IDEMPOTENCY_CONFLICT`

---

## Roles

| Role | Core permissions |
| --- | --- |
| Customer | Own profile, addresses, cart, bookings, payments, reviews, support tickets |
| Professional | Own onboarding, availability, assignment responses, execution, earnings |
| Operations Manager | Read/update bookings, assign professionals, handle exceptions, contact parties |
| Support Executive | Read assigned support context, message users, apply permitted resolutions |
| Finance Manager | Read payments, reconcile, approve refunds, process settlements |
| Catalog Manager | Catalog, pricing, zones, offers, availability settings |
| Super Admin | Roles, permissions, platform configuration, high-risk operations |

## Business rules

| ID | Rule |
| --- | --- |
| BR-001 | **Price authority** — backend computes all totals from versioned pricing and customer context |
| BR-002 | **Historical integrity** — terminal bookings keep the exact price/policy snapshot used at purchase |
| BR-003 | **Slot ownership** — a hold belongs to one customer/session and is non-transferable |
| BR-004 | **Conflict protection** — no overlapping assigned/active bookings after travel buffers |
| BR-005 | **Assignment eligibility** — active, approved, skilled for every service, serviceable for the address |
| BR-006 | **OTP override** — authorized roles only, with reason and audit event |
| BR-007 | **Cancellation policy** — fee depends on actor, timing, travel state, start state, policy version |
| BR-008 | **Refund limit** — cumulative successful + pending refunds ≤ captured payment |
| BR-009 | **Review eligibility** — only the customer on a completed booking |
| BR-010 | **Settlement freeze** — disputed/refunded/chargeback earnings stay unavailable until resolved |
| BR-011 | **Data scope** — admin access limited by permission and optional city/region scope |
| BR-012 | **Deletion** — financial records retained; user deletion via deactivation/anonymization/soft delete |

---

## Scope

### P0 — MVP

Auth with JWT access + rotating refresh · addresses and serviceability · catalog with variants,
add-ons and city/zone pricing · cart, coupons, server-calculated totals, checkout · availability,
slot holds, conflict-safe confirmation · payment orders, webhook verification, reconciliation ·
professional onboarding, skills, zones, availability · automatic/manual assignment with timeout and
retry · real-time status + notification centre · start/completion OTP, checklist, evidence · booking
history, cancellation, ratings, basic support · admin live operations, booking/professional/catalog
management · audit logging, Docker, API docs, tests for critical flows.

### P1 — marketplace completeness

Rescheduling with atomic slot transfer · partial/full refunds with timeline and retries · wallet,
commission, settlements, payouts · memberships, constrained coupons, customer credits · support SLAs
and escalation · professional scoring, incentives, penalties, suspension · reporting, exports,
finance reconciliation dashboards.

### P2 — later

Native apps, live GPS tracking, route optimization · demand-based dynamic pricing · recommendations
and bundles · automated KYC provider integration · fraud/risk scoring and chargeback management ·
multi-language and multi-country taxation.

### Out of scope

Copying any competitor's design, copy, private APIs or data · clinical or regulated healthcare
workflows · multi-branch cash reconciliation · microservices before a validated need · fully
automated settlement without administrative control.

---

## Non-functional targets

| Area | Target |
| --- | --- |
| Availability | 99.5% monthly, excluding planned maintenance |
| API latency | p95 < 500 ms reads, < 900 ms writes (excluding third-party time) |
| Scalability | Stateless API instances; jobs and sockets coordinate through shared Redis |
| Consistency | Transactions or idempotency on booking, acceptance, payment, refund, settlement |
| Security | OWASP-aligned controls, dependency scanning, secret management, secure headers |
| Accessibility | WCAG 2.2 AA on customer and professional critical flows |
| Compatibility | Latest two versions of Chrome/Edge/Firefox/Safari; responsive from 360 px |
| Observability | Structured logs, request ids, error tracking, metrics, alerts on critical failures |
| Maintainability | Strict TypeScript, module boundaries, lint/format, CI-enforced tests |

---

## Testing

| Layer | Scope | Tools |
| --- | --- | --- |
| Unit | Pricing, validators, transition rules, assignment scoring, cancellation/commission logic | Vitest |
| API integration | Auth, booking transactions, concurrency, permissions, webhooks, refunds | Supertest + test PostgreSQL |
| Component | Forms, slot picker, price breakdown, tables, permission states | React Testing Library |
| End-to-end | Booking, acceptance, OTP completion, cancellation, admin recovery | Playwright |
| Load | Slot holds, concurrent acceptance, webhook bursts, reports | k6 |
| Security | Dependency + secret scanning, authz checks, input and upload attacks | CI scanners + checklist |

### Acceptance scenarios that must pass

| ID | Scenario |
| --- | --- |
| AC-01 | Two customers race the last slot capacity — exactly one booking is created |
| AC-02 | Two professionals accept the same request concurrently — exactly one is assigned |
| AC-03 | A verified webhook is delivered twice — payment and booking confirm once |
| AC-04 | Frontend callback arrives before webhook — UI shows pending verification, never false-confirms |
| AC-05 | Hold expires before payment — creation fails `SLOT_UNAVAILABLE` and capacity is released |
| AC-06 | Invalid transition or cross-customer access — denied |
| AC-07 | Professional missing a skill or zone — excluded, and not manually assignable either |
| AC-08 | Completion retried after a timeout — invoice and wallet entries created once |
| AC-09 | Partial refund plus a later refund cannot exceed the captured amount |
| AC-10 | Support user exceeds their refund limit — rejected and audited |
| AC-11 | Reschedule fails after the new hold — the original booking slot survives intact |
| AC-12 | Every P0 requirement maps to at least one automated or documented test |

### Definition of done

Acceptance criteria met · API contract and validation documented · backend permission check **and**
matching frontend state · loading/empty/error/success states implemented · CI green with no critical
accessibility regression · logs, metrics and audit events present · docs and demo seed data updated.

---

## Phase 1 — what we're picking

Phase 1 is a **vertical slice**, not a horizontal foundation layer. The goal is one working path —
`DRAFT → SLOT_HELD → PAYMENT_PENDING → CONFIRMED → ASSIGNMENT_PENDING → ASSIGNED` — for one city, one
category and three services, driven by tests and curl. No UI polish.

Rationale: the concurrency core (AC-01, AC-02, AC-03, AC-05) is the part that is hard to get right
and hard to retrofit. Building it first proves the product works; building auth and catalog CRUD
first proves nothing.

### In scope

- Monorepo, Docker Compose (PostgreSQL + Redis), CI, strict TypeScript, lint and format gates.
- Migrations and schema for **identity, catalog and commerce only** — not all twelve domains.
- Auth: JWT access + rotating refresh with token families, HTTP-only cookies, rate limiting
  (AUTH-FR-001…005, 007).
- RBAC tables and middleware, but only three seeded roles: `customer`, `professional`, `super_admin`.
- Booking state machine as a pure domain module — transition map, `booking_status_history` written in
  the same transaction, unit tested before any booking UI exists.
- Availability, slot holds with Redis-backed expiry, and conflict-safe booking confirmation.
- Payment order creation and webhook signature verification against a sandbox gateway.
- Automatic assignment with time-limited offers, atomic acceptance and retry on timeout.
- AC-01, AC-02, AC-03 and AC-05 as integration tests against a real PostgreSQL instance.

### Foundations to get right now, because retrofitting them is expensive

- Money as integer minor units behind a single `Money` type — never a bare `number`.
- Standard error envelope, request-id propagation and schema validation at the edge.
- OpenAPI generated from route schemas and validated in CI.
- `status` as a private field of the booking domain service, so no controller can write it directly.
- Outbox table, even if its only consumer is a log line at first.
- An idempotent, versioned seed script. Every acceptance scenario needs a specific fixture state, and
  the demo is the deliverable.

### Deliberately deferred

Socket.IO (poll on an interval; sockets land in Milestone 5) · object storage and KYC uploads ·
email/SMS/push · wallet and settlements · coupons and memberships · review moderation · admin exports
and reporting · the `operations`, `support`, `finance` and `catalog` roles, which arrive with the
admin app.

### Decisions Phase 1 needs (proposed — pending sign-off)

| ID | Proposal | Reason |
| --- | --- | --- |
| OD-01 | Online payment only; cut pay-after-service from MVP | Doubles the payment branches of the state machine and adds a collections problem that cannot be demonstrated |
| OD-02 | Automatic assignment only | Customer-selected professionals push eligibility filtering into browse-time queries — a much larger surface |
| OD-03 | One city, 2–3 zones, flat travel fee | Zone-varying travel fees add a pricing dimension with no demo value |
| OD-04 | Razorpay | INR and India per A-01, with sandbox webhook replay for real signature verification |
| OD-11 | Capacity-based availability; per-professional conflict enforced only at acceptance | See below |

**On OD-11.** Assignment happens *after* `CONFIRMED`, so no professional is chosen at slot-hold or
booking time. Availability therefore cannot be a per-professional conflict check — it must be
capacity: how many eligible professionals are free in that zone and window. Two consequences:

- Do **not** materialize a slots table. Derive availability from `professional_availability`, existing
  bookings and travel buffer at query time; `slot_holds` decrement derived capacity; the hard
  integrity guarantee (BR-004) binds at *acceptance* via an exclusion constraint on the professional's
  time range.
- A booking can be confirmed and paid but unassignable — which is exactly why `ASSIGNMENT_FAILED`
  exists. Phase 1 must choose a policy. Proposed: auto-refund after N failed attempts and notify,
  since a demo has no operations team to work the queue.

---

## Milestones

| # | Weeks | Deliverables |
| --- | --- | --- |
| 1 — Foundation | 1–2 | Repo, Docker, CI, schema, auth, sessions, RBAC, shared UI and API error standards |
| 2 — Catalog & customer core | 3–4 | Location, addresses, catalog, pricing, cart, quote, profile |
| 3 — Booking & payment | 5–7 | Availability, holds, booking transaction, payment orders, webhooks, history |
| 4 — Professional ops | 8–10 | Onboarding, KYC, skills, availability, assignment, status, OTP execution |
| 5 — Admin & real-time | 11–12 | Ops queues, admin management, Socket.IO, notifications |
| 6 — Quality & deployment | 13–14 | Reviews, support, audit, E2E, security hardening, seed, deploy |
| 7 — P1 expansion | Post-MVP | Rescheduling, refunds, wallet, settlements, promotions, memberships, analytics |

### Environments

| Env | Purpose | Controls |
| --- | --- | --- |
| Local | Development and tests | Docker Compose, seeded data, mail/payment mocks |
| Staging | Integrated QA and demo rehearsal | Production-like config, sandbox gateways, anonymized seed |
| Production / Demo | Public deployment | Managed secrets, HTTPS, backups, monitoring, rate limits, restricted admin |

---

## Conventions

- **Requirement IDs** (`AUTH-FR-001`, `BOOK-FR-004`, `BR-007`, `AC-03`) go in branch names, commit
  messages, PR descriptions and test names. A PR should say which IDs it closes.
- Priorities: **P0** ships in MVP, **P1** completes the marketplace, **P2** is an enhancement.
- A requirement is done only when its behaviour is demonstrated by an automated or documented test.
- The server is authoritative for price, availability, booking status, payment state, permissions and
  refunds. Anything a client sends about those is an input to validate, never a fact to trust.

## Open decisions

These block or reshape work downstream and need owners before the milestones they touch:

| ID | Decision | Blocks |
| --- | --- | --- |
| OD-01 | Pay-after-service in MVP, or online payment only? | Milestone 3 |
| OD-02 | Customer-selected professional, or automatic assignment only? | Milestone 4 |
| OD-03 | Initial city, zone and travel-fee model | Milestone 2 |
| OD-04 | Which payment gateway for the deployment region | Milestone 3 |
| OD-05 | Mandatory KYC documents and retention period | Milestone 4 |
| OD-06 | Cancellation windows and fees per lifecycle state | Milestone 3 |
| OD-07 | When the professional app sees customer location/contact | Milestone 4 |
| OD-08 | Settlement frequency and minimum payout threshold | P1 |
| OD-09 | Do bundles allow multiple professionals or require one multi-skilled? | Milestone 4 |
| OD-10 | Analytics provider, error tracker, deployment platform | Milestone 1 |
| OD-11 | Capacity model for availability, and the policy for a confirmed-but-unassignable booking | Phase 1 |

## Demo data

Cities: Gurugram, Delhi, Noida, Bengaluru. Categories: Salon for Women, Salon for Men, Spa and
Massage, Hair Care, Skin Care, Nail Care. Seed ≥20 services with variants/add-ons/durations and
location prices, 10 customers, 12 professionals across skills and KYC states, at least one booking
in every primary and exception state, one admin user per role, payments in every state, and support
tickets across the queue.

Demo script: four accounts (customer, professional, operations manager, super admin) walking
booking → payment verification → assignment → OTP execution → review → cancellation/refund → audit
history.

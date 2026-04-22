---
name: outegram-market-milestones
description: Reference and execution guide for the Outegram Marketplace Platform milestone roadmap (Laravel 13, 9 milestones M0-M8, 57 issues). Use when the user mentions outegram, market, milestone-issues.md, any MX-YY issue id (e.g. M1-03, M3-02), branch names like feature/m2-05-messaging or chore/m0-03-ci-pipeline, milestone gates, architectural rules for Atomic Actions/Orchestrators/ledger immutability, or asks about marketplace platform implementation, seller onboarding, escrow clearing, Stripe webhooks, Elasticsearch search sync, or Filament admin work scoped to this project.
---

# Outegram Marketplace — Milestone Execution Guide

This skill drives any work that touches the `outegram/market` milestone roadmap. The authoritative document is `docs/process/MILESTONE_ISSUES.md` in the repo (mirrored offline at [reference.md](reference.md)).

**Project:** Laravel 13 marketplace (sellers list services, buyers order, Stripe escrow, Elasticsearch search, Filament admin).
**Scope:** 9 milestones (M0 → M8), 57 issues total. Each issue = one GitHub Issue, one branch, one PR.

## When to apply this skill

Apply automatically whenever the user:

- References an issue id matching `M[0-8]-[0-9]{2}` (e.g. `M1-03`, `M3-02`, `M7-01`).
- References a branch matching `(feature|chore|db)/m[0-8]-[0-9]{2}-*`.
- Mentions "milestone", "gate condition", "architecture-exception", or `milestone-issues.md`.
- Asks to implement, scope, review, or plan any piece of the marketplace roadmap.

## Milestone map

| Milestone | Name | Issues | Opens when |
|---|---|---|---|
| M0 | Infrastructure & Architecture | 8 | Start of project |
| M1 | User + Taxonomy + Service | 10 | M0 complete + contract signed |
| M2 | Orders + Messaging + Reviews | 9 | M1 complete |
| M3 | Payments + Billing | 7 | M2 complete **and** zero open `architecture-exception` |
| M4 | Notifications + Analytics | 5 | M3 complete |
| M5 | Search | 5 | M1 complete (may run parallel to M2–M4) |
| M6 | Admin Panel | 5 | M3 complete |
| M7 | Stabilization | 5 | M6 complete |
| M8 | Production Launch | 3 | M7 complete + security audit signed off |

For full issue text — branch, tasks, acceptance criteria, dependencies — consult [reference.md](reference.md) and search for the exact `MX-YY` id.

## Workflow — implementing a single issue

When the user asks to work on an issue (e.g. "implement M1-03"), follow this loop:

1. **Open [reference.md](reference.md)** and locate the issue heading `### MX-YY — ...`. Read Objective, Tasks, Acceptance criteria, and the `Depends on:` line in full before writing code.
2. **Verify dependencies are merged.** If the `Depends on:` issue is not done, stop and tell the user. Do not start work on a blocked issue.
3. **Verify the milestone gate.** If the milestone hasn't opened (see table above), stop and surface the gate condition.
4. **Create the branch** using the exact name from reference.md:
   - M0: `chore/m0-XX-*` (except shared-kernel / reference-implementations use `feature/`)
   - M1: `db/m1-01-*`, `db/m1-02-*`, otherwise `feature/m1-XX-*`
   - M2: `feature/m2-XX-*` except `chore/m2-09-observability`
   - M3: `feature/m3-XX-*` except `chore/m3-06-ledger-audit` and `chore/m3-07-webhook-reliability`
   - M4: mixed — see reference.md
   - M5: `feature/m5-XX-*`
   - M6: `feature/m6-XX-*`
   - M7, M8: `chore/m7-XX-*`, `chore/m8-XX-*`
5. **Work tasks in listed order.** The checklist in reference.md is ordered deliberately — earlier tasks are prerequisites for later ones in the same issue.
6. **Respect the architectural rules** in [Architectural rules](#architectural-rules) below.
7. **Write Pest tests matching each bullet** in the issue's Pest section. Acceptance criteria are verification targets, not suggestions.
8. **Open the PR** using the template in [PR conventions](#pr-conventions).

## Architectural rules

These rules are enforced by CI (custom PHPStan rules from M0-04) and by reviewers. Violating them fails `architecture-rules` job in `.github/workflows/ci.yml`.

### Actions — Atomic vs Orchestrator

- **Atomic Action** (`app/Domain/{Domain}/Actions/Atomic/`): does exactly one thing against one table or one external service. Must NOT:
  - Instantiate or resolve another Action class.
  - Call `dispatch()` on a Job.
  - Call `event()` or `Event::dispatch()`.
  - Call `DB::transaction()`, `DB::beginTransaction()`, or `DB::commit()`.
- **Orchestrator Action** (`app/Domain/{Domain}/Actions/Orchestrators/`): composes Atomic Actions inside a single `DB::transaction()`, then dispatches events/jobs **after** commit.
- Reference implementations to measure new Actions against: `RegisterUserAction`, `CreateServiceAction`, `PlaceOrderAction` (all in M0-08, marked `// REFERENCE IMPLEMENTATION`).

### Financial tables — immutability

- `ledger_entries` and `admin_actions` tables must have NO `updated_at` column and NO soft deletes. They are append-only audit trails.
- All financial state mutations (payment, escrow, clearing, refund, withdrawal, credit) write a LedgerEntry inside the same DB transaction that mutates state.
- Seller balance is always computed from ledger: `sum(credits) - sum(debits)`. Never stored as a mutable field.
- `php artisan ledger:reconcile` must pass before M4 opens, and daily in production.

### Money

- All monetary amounts are integer cents in a `Money` value object (`app/Shared/ValueObjects/Money.php`). No float arithmetic anywhere in fee, tax, escrow, or payout calculations.

### Events

- All queued listeners for `*NotificationEvent` events must declare `public bool $afterCommit = true` so they fire only after the transaction commits.

### Webhooks

- Every Stripe webhook handler is idempotent — receiving the same `event_id` twice must produce exactly one state change.
- Webhook signature verification runs before any handler logic. Invalid signature → HTTP 400.

### Queue priorities (from M4-03)

| Queue | Workers | Use for |
|---|---|---|
| `critical` | 1 | Payment webhooks, fraud flags, account bans |
| `high` | 2 | Order notifications, OTP, onboarding review |
| `default` | 3 | General jobs (requires justifying comment if used) |
| `low` | 2 | Search indexing, rating recalc, seller stats |
| `batch` | 1 | Email digests, reports, full reindex |

Every Job class must have an explicit queue assignment.

### Search

- MySQL is never queried during `GET /api/v1/search/services`. Search reads exclusively from Elasticsearch.
- Index changes use versioned aliases (`service_listings` → `service_listings_v1`) so reindexing can happen without downtime.

## PR conventions

Every PR for an issue must include:

1. **Title:** `MX-YY — <issue name>`
2. **Body sections:**
   - `## Summary` — what was built, in 2–4 bullets.
   - `## Acceptance criteria` — copy the issue's Acceptance criteria verbatim, each preceded by `- [x]` when satisfied.
   - `## Verbal explanation test` (M0-08 / reference-implementation PRs only) — the author summarises in plain English why each Action lives where it does.
   - `## Architecture notes` — flag any architectural exceptions or deviations. If this section is non-empty, an `architecture-exception` GitHub issue must be opened and linked.
3. **Checks that must be green before merge:** `install`, `lint`, `analyse`, `test`, `docs-freshness`, `architecture-rules`.
4. **Reviewer:** the role defined in `.github/CODEOWNERS` for the touched paths. Payment-critical PRs (`payment-critical` label) require the Architecture Steward.

## Milestone gate checks

Before declaring a milestone complete:

- All its issues are closed and their PRs merged to `main`.
- CI on `main` is green on the merge commit.
- For M3: zero open issues with label `architecture-exception` (hard gate — no exceptions allowed from M3 onward).
- For M0: Technical Approver has committed sign-off to `docs/architecture/milestone-0-execution-contract.md`.
- For M7: Architecture Steward sign-off on `docs/process/security-audit-m7.md`.
- For M8: Technical Approver **and** Architecture Steward both comment "Go for launch" on the M8-01 issue.

## Quick issue lookup

When asked "what is MX-YY?", respond with:

1. The issue title.
2. Its branch name.
3. Its `Depends on:` line.
4. A one-sentence Objective.
5. Offer to read the full Tasks/Acceptance from [reference.md](reference.md).

Do not paraphrase acceptance criteria — quote them exactly when the user asks.

## Anti-patterns

- Do not invent issue ids. If the user references `MX-YY`, verify it exists in reference.md before proceeding.
- Do not merge a PR with an unresolved `@architecture-exception` comment once M3 has opened.
- Do not set `users.seller_level` directly — only `SellerLevelCalculator` (M2-08) may change it.
- Do not add `updated_at` or soft deletes to `ledger_entries` or `admin_actions`.
- Do not call `DB::transaction()` inside an Atomic Action.
- Do not add a file under `Actions/Atomic/` that dispatches a job or event.

## Additional resources

- Full issue text, tasks, and acceptance criteria: [reference.md](reference.md)
- Live source of truth (may drift from reference.md): https://github.com/outegram/market/blob/main/milestone-issues.md

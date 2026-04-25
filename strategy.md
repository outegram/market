# Freelance Marketplace Platform — Architecture & Execution Strategy

**Document type:** Internal Engineering Strategy
**Audience:** CTO, Tech Leads, Senior Laravel Engineers, PMs
**Status:** Draft v1.0
**Stack:** Laravel 13 · PHP 8.3 · MySQL 8 · Redis · Elasticsearch 8 · Reverb · Horizon · Stripe Connect · Livewire 4 · FluxUI
**Architecture:** Modular Monolith (Lean Modular) with DDD-inspired module boundaries
**Scope:** Buyer/Seller app only. Admin Panel is a separate application and is out of scope.

---

## 0. Executive Summary

This document is the single source of truth for building a Fiverr-class two-sided freelance marketplace. It is opinionated on purpose. Every decision here exists to prevent a predictable failure mode we have seen break marketplaces: double charges, lost deliveries, Elasticsearch drift, refund abuse, seller impersonation, race conditions on order state, and the slow collapse of a codebase into a "big ball of mud" because module boundaries were suggestions instead of rules.

The platform is a **modular monolith**, not microservices. Microservices on day one kills startups. A disciplined modular monolith with enforced boundaries, event-driven inter-module communication, and a clean shared kernel gives us the scaling runway for years and the option to extract services later (Search, Messaging, Payments are the likely first extractions).

The hardest problems are not the CRUD. They are:

1. **Money correctness** — every cent must be auditable. No floats, ever. Ledger-first accounting.
2. **Order state machine integrity** — one invalid transition and you have a support ticket that lasts weeks.
3. **Search freshness vs consistency** — Elasticsearch will drift from MySQL; we must accept this and design for it.
4. **Message abuse and contact-info leakage** — sellers will try to move deals off-platform. This is an existential revenue threat.
5. **Idempotency everywhere** — webhooks retry, users double-click, queues replay.

The rest of this document is how we solve those problems specifically, not generically.

---

# PART 1 — FULL FIVERR SYSTEM ANALYSIS

This section is a reverse-engineered mental model of how Fiverr works at an engineering level. We are not guessing at pixels; we are inferring the systems that must exist to explain the observed behaviour.

## 1.1 Buyer-side experience

A buyer's journey has four distinct system interactions: **discover → evaluate → transact → consume**.

**Discover** is search and recommendation. The homepage and category pages are driven by a recommender that mixes personalization signals (category views, past orders, location, language) with editorial boosting (featured gigs, Pro gigs, seller level weighting). Search itself is Elasticsearch-backed with heavy use of function_score queries: base relevance (BM25 on title/tags/description) multiplied by signals like `seller_level`, `response_rate`, `completion_rate`, `recent_orders_count`, and a freshness decay. The autocomplete is a separate index (edge n-grams or completion suggester) for sub-100ms latency.

**Evaluate** is the gig page. This page is heavily cached — gig content changes rarely relative to how often it is viewed. The packages (Basic/Standard/Premium), FAQs, portfolio images, seller profile snippet, and reviews are all read-heavy. Reviews are paginated and sorted by relevance (likely a helpfulness score, not just recency).

**Transact** is the checkout. The buyer picks a package, optional add-ons (gig extras — extra revision, extra fast delivery, additional concept), and enters requirements. Payment is authorized via Stripe (cards, PayPal, Apple Pay, local methods). The money is captured but held — this is the escrow illusion. Fiverr is not a real escrow (they don't hold client funds in trust accounts; they use Stripe Connect's delayed payout mechanism). The buyer is charged immediately, the seller is paid only on order completion + a clearance period.

**Consume** is the order page — the central artifact of the platform. It contains the conversation thread, the delivery files, the revision counter, the timer, and the action buttons. State transitions here drive almost every notification in the system.

## 1.2 Seller-side experience

A seller has three modes: **present → work → get paid**.

**Present** is the profile + gig portfolio. A seller creates up to N gigs (Fiverr caps this by level — new sellers get ~7). Each gig has: title, category, subcategory, service type, search tags (max ~5), three packages with price/delivery/revisions/features, FAQ, requirements questions, gallery (images + video + PDF), and SEO metadata.

**Work** is the order inbox and the message inbox. These are deliberately separate: orders have contractual state and timers, messages do not. A seller sees a queue of active orders sorted by delivery deadline. Late deliveries penalize their metrics severely (on-time delivery rate directly affects level retention and search ranking). They can also send "custom offers" from inside a message thread — this creates an unpaid pending order that becomes real on buyer acceptance + payment.

**Get paid** is the earnings page. Funds move: `Escrow (pending) → Available for withdrawal (after 14-day clearance) → Withdrawn`. Withdrawal methods are Payoneer, PayPal, bank transfer — implemented via Stripe Connect's Express/Custom accounts or direct Payoneer API. Fiverr's cut is 20% flat on the seller side.

## 1.3 Gigs — creation and management

Gig creation is a multi-step wizard with heavy validation:

1. **Overview:** title, category, subcategory, service type, tags, metadata (buyer-facing filters like "language", "framework", "style"). The metadata is category-dependent — logo design has "style" and "file format", writing has "word count" and "language".
2. **Pricing:** three-tier package matrix. Each package has title, description, delivery days, revisions, and a feature checklist. Gig extras are separate add-ons with their own price + delivery impact.
3. **Description & FAQ:** long-form markdown-like editor. FAQs are structured Q&A pairs.
4. **Requirements:** structured questionnaire the buyer must answer on order. Each question has type (text, multiple choice, file), required flag, and appears on the order page.
5. **Gallery:** images (with dimension/aspect-ratio requirements), up to one video (with length cap), PDFs. Files go through image processing (resize, WebP conversion), virus scanning, and likely AI moderation (NSFW/stolen-content detection).
6. **Publish:** queued for manual + automated review. The gig sits in `pending_approval` until approved. Once live, edits to certain fields (price, packages) trigger re-review.

Inferred engineering decisions:
- Gigs are soft-deleted, never hard-deleted (review history, past orders, SEO backlinks).
- Gig versions exist — edits create a new draft version while the live version continues serving orders.
- Elasticsearch indexing is async, triggered by domain events (`GigApproved`, `GigUpdated`, `GigPaused`).

## 1.4 Search

Search is the single most important system for GMV. Fiverr's search behaves like this:

- Free-text search with typo tolerance (ES `fuzziness: AUTO`).
- Category + subcategory filters (faceted).
- Price range, delivery time, seller level, seller language, seller country, online-now.
- Sort modes: Best selling, Recommended (default), Newest.
- "Recommended" is the money sort. It is a function_score composition: text relevance × seller quality score × gig popularity score × freshness decay × personalization boost.
- Personalization likely uses the user's browsing history (recent category views) and past purchases as signals re-ranked on top of the base result set (not as primary ES query clauses — that would be expensive).

Inferred: search index is denormalized heavily. A single `gigs` document contains seller-level fields (level, country, response rate) copied in at index time. When seller attributes change, all their gigs must be re-indexed — this is a background job triggered by `SellerMetricsRecalculated` events.

## 1.5 Filters

Filters are both query-level (ES terms/range queries) and aggregation-driven (the "N available" counts next to each filter option). Aggregations are expensive; Fiverr likely caches filter counts per category+subcategory for 5-15 minutes and only runs live aggs for the dynamic subset.

## 1.6 Order lifecycle

See Part 4 for the full map. Fiverr's observable order states are roughly:

`Pending Requirements → In Progress → Delivered → Revision Requested → Redelivered → Completed → (Reviewed)`

Plus terminal/off-path: `Cancelled (Mutual / Unilateral) · Disputed · Refunded · Late`.

The key insight: **the delivery deadline countdown pauses** when the seller is waiting on buyer requirements or when a revision is in flight. This is a state-aware timer, not a wall clock — implementation requires storing the remaining seconds at each pause and resuming with a new `resumes_at` when work restarts.

## 1.7 Messaging

Messaging is a first-class system, not a chat widget bolted on. Features inferred:

- Threads are 1:1 between buyer and seller, plus "system messages" (order events injected as messages by the platform: "Order delivered", "Revision requested").
- Messages support attachments (files, images, audio notes).
- There is automated scanning for contact info leakage (emails, phone numbers, Telegram handles, WhatsApp links) and external payment discussion. Violations get the message flagged, redacted, or the user warned/suspended.
- Custom Offers are a structured message type — the sender creates an offer (price, delivery days, description, revisions) that renders as a card with "Accept" / "Decline" buttons. Accept triggers an order flow.
- Read receipts exist (seller response rate depends on them).
- Online/offline presence is shown — Reverb/Pusher-style websocket for this in our stack.

## 1.8 Custom offer requests

Buyers can send a "Buyer Request" (brief + budget + deadline) that sellers respond to. This is a lightweight inbox system for sellers to find work proactively. Rate-limited on both sides (buyers get N requests/day, sellers get N responses/day) to prevent spam. This is a separate module from messaging — the brief is a first-class entity with its own lifecycle.

## 1.9 Milestones

Milestones exist on large orders. A big order is split into phases, each with its own deliverable, deadline, and partial payment release. Implementation inference: a milestone is a child entity of an order, each with its own state machine that mirrors the parent order (requirements → in progress → delivered → approved). The parent order is "complete" only when all milestones are approved. Payment escrow releases per-milestone.

## 1.10 Delivery

Delivery is a structured action: seller uploads final files + writes a delivery message, clicks "Deliver Now". This:
- Changes order state to `Delivered`.
- Starts a 3-day auto-complete timer (configurable — Fiverr uses 3 days).
- Stops the delivery deadline timer.
- Sends `OrderDelivered` event → notifies buyer, updates seller metrics provisionally.

The buyer has three actions: accept (→ Completed), request revision (→ Revision Requested), or raise an issue (→ Resolution Center).

## 1.11 Revisions

Revisions are counted. The package defines N revisions included. After N, additional revisions require either a gig extra purchase or seller goodwill. Each revision cycle:
- Pauses the auto-complete timer.
- Resets some metric calculations (the seller's delivery timer does NOT restart from scratch — they get a configured extension, often 1-3 days).
- Is logged as a revision event on the order.

## 1.12 Disputes

The Resolution Center is a structured dispute module. Flow:
1. Either party opens a resolution with a reason (late delivery, non-delivery, low quality, buyer unresponsive, etc.) and a proposed solution (cancel, partial refund, extend time).
2. The other party has 48 hours to accept or counter.
3. If accepted, action executes automatically (refund, cancel, extension).
4. If rejected or ignored past 48h, the case escalates to Customer Support (a human operator in Fiverr's admin panel — irrelevant to our current scope but we must emit the escalation event).

## 1.13 Cancellations

Three flavours:
- **Mutual cancellation**: both parties agree via Resolution Center → full refund to buyer, no penalty to seller's stats.
- **Seller-initiated (late/unable)**: full refund, negative impact on seller metrics (order completion rate drops, affects level).
- **Admin cancellation (TOS violation, fraud)**: full refund, severe penalty on offending party.

The critical business rule: a cancelled order still exists in the database with a `cancelled` state; it is NEVER deleted. All money movement is recorded in the ledger as reversing entries.

## 1.14 Refunds

Refunds in Fiverr go to the buyer's **Fiverr balance** by default (not back to card) to reduce Stripe refund fees and to keep money inside the ecosystem. Buyers can request a refund to the original payment method, which is processed but takes 5-10 days. This is a deliberate UX + financial decision we should replicate.

## 1.15 Notifications

Multi-channel: in-app (real-time via Reverb), email, push (mobile), SMS for critical events. Each notification type has a user preference (email on/off, push on/off per category). The critical engineering decision: notifications are driven by **domain events**, not inlined in controllers. Every state transition emits an event; notification listeners subscribe and fan out to channels.

## 1.16 Reviews

Reviews are submitted by the buyer post-completion. Rating is 1-5 stars across multiple axes (communication, service as described, buy again) + a public text review. Sellers can respond publicly to a review (one response, non-editable after 24h). Reviews are immutable after a grace period. Fake/abusive reviews are a constant problem — the system must detect review manipulation (reviewing own gigs via sockpuppet, review swaps, incentivized reviews).

## 1.17 Abuse prevention

Key vectors we must defend:

- **Off-platform payment**: sellers asking buyers to pay on PayPal/bank directly. Mitigated via message content scanning + user reporting + immediate suspension on confirmed violation.
- **Account sharing/reselling**: seller sells an account with good stats. Mitigated via device fingerprinting, IP analysis, KYC for withdrawals.
- **Review manipulation**: fake 5-stars from sockpuppets. Mitigated via buyer account quality signals (account age, payment history, device uniqueness) weighted into review visibility.
- **Refund abuse**: buyer orders, gets delivery, claims non-delivery, refunds. Mitigated via buyer trust score + requiring dispute evidence + cap on refund rate per buyer.
- **Seller impersonation / fake portfolios**: stolen work in gig galleries. Mitigated via reverse image search + DMCA system + seller ID verification.
- **Gig scraping/clone**: a seller copies another seller's gig wholesale. Mitigated via similarity detection on gig creation.

## 1.18 Trust system

Trust is expressed through:
- **Seller levels**: New Seller → Level 1 → Level 2 → Top Rated Seller. Automatic promotion based on 60-day rolling metrics (orders completed, rating, on-time delivery, response rate, warnings count). Automatic demotion on metric decay.
- **Verified badges**: ID verified, payment verified, phone verified.
- **Pro program**: manually curated high-end sellers with separate branding.
- **Buyer trust score**: not visible, but internal. Affects refund eligibility, dispute priority, message rate limits.

## 1.19 Account restrictions

Graduated enforcement:
- **Warning** (internal record, user notified).
- **Feature restriction** (can't create new gigs, can't send messages, can't withdraw).
- **Temporary suspension** (account disabled for N days).
- **Permanent ban** (account closed, balance forfeited on confirmed fraud, otherwise balance paid out minus owed refunds).

Every restriction is an event with an actor (system/moderator), reason, duration, and affected features.

## 1.20 Seller levels

See 1.18. Implementation is a scheduled job (nightly) that recomputes each seller's metrics over the rolling window and applies level changes. Level changes emit events that trigger re-indexing (level is a search ranking input) and notifications.

## 1.21 Featured gigs / promotions

Sellers pay to promote gigs (Fiverr calls this "Seller Plus Promoted Gigs" — essentially per-click advertising inside search results). Implementation:
- Budget-per-day + CPC bid.
- Served as a separate slot in search results (not mixed into organic — clearly labelled "Ad").
- Click tracking with fraud detection (rate-limit clicks per user, bot detection).
- Budget depletion pauses the campaign mid-day.

This is a v2 feature for our roadmap — not Milestone 1-9 scope.

## 1.22 Coupons / promotions

- **First-order discounts** for buyers (acquisition).
- **Category/seasonal promotions** (platform-funded).
- **Seller-specific coupons** (Seller Plus feature).
- **Referral credits**.

Engineering: coupons are a flat module with code, type (fixed/percent), scope (category/seller/global), usage caps (per-user, total), min order value, date range, and stacking rules. A single well-specified module prevents the usual sprawl of "campaign" tables.

## 1.23 Recommendations

Three layers:
1. **Homepage personalization**: recently viewed, categories browsed, "recommended for you" based on collaborative filtering on the user's order/view history.
2. **Gig page "related gigs"**: content-based similarity (same category + tag overlap + price tier + language).
3. **Post-purchase**: "Buyers also ordered" — basket analysis across completed orders.

For v1, a simple content-based recommender + "top sellers in category" is enough. CF/embedding-based recommendation is a v2/v3 investment.

---

# PART 2 — SYSTEM MODULE BREAKDOWN

## 2.0 Module architecture principle

We use a **modular monolith** organized under `app/Modules/{ModuleName}`. Each module is a bounded context. Inter-module communication is allowed ONLY via:

1. **Domain events** (preferred — via Laravel events, with async listeners on queues).
2. **Published contracts** (`Contracts/` folder inside each module — interfaces the rest of the app may use).
3. **Shared Kernel** (small, stable, versioned).

**Forbidden:** directly importing another module's internal classes (Models, Actions, Services that are not in `Contracts/`). Enforced via Deptrac or `PHPStan` + custom rules in CI.

Suggested folder layout per module:

```
app/Modules/Orders/
├── Contracts/              # Public interfaces — other modules may depend on these
├── Domain/
│   ├── Models/             # Eloquent models (Order, OrderItem, OrderEvent)
│   ├── Enums/              # OrderStatus, CancellationReason
│   ├── Events/             # OrderCreated, OrderDelivered, ...
│   ├── Exceptions/
│   └── ValueObjects/       # Money, OrderReference
├── Application/
│   ├── Actions/            # CreateOrderAction, DeliverOrderAction
│   ├── Queries/            # GetActiveOrdersQuery (read-side)
│   └── Services/           # OrderStateMachine
├── Infrastructure/
│   ├── Policies/
│   ├── Listeners/
│   ├── Jobs/
│   └── Repositories/       # Only if needed
├── Http/
│   ├── Controllers/
│   ├── Requests/
│   ├── Livewire/
│   └── Resources/
├── Database/
│   ├── Migrations/
│   ├── Factories/
│   └── Seeders/
├── routes/
│   └── web.php
├── Tests/
│   ├── Feature/
│   └── Unit/
└── OrdersServiceProvider.php
```

Every module registers its own service provider. The monolith's `config/app.php` (or a `config/modules.php` registry) lists the active modules.

---

## 2.1 Identity & Authentication

**Responsibility:** Registration, login, password reset, 2FA, email verification, phone verification, social login, session management, device tracking, account linking (buyer+seller are a single user with role grants, not separate accounts).

**Boundaries:** Owns the `users`, `sessions`, `devices`, `two_factor_tokens`, `email_verification_codes` tables. Exposes `UserContract` interface (id, email, roles, is_verified).

**Dependencies:** Shared Kernel only.

**Risks:**
- Credential stuffing attacks — mitigate with rate limiting, reCAPTCHA on signup, password breach check (HIBP API).
- Account takeover via email compromise — require re-auth + 2FA for payout method changes.
- Multi-account abuse — device fingerprinting + IP clustering.

**Technical challenges:**
- Unified buyer/seller account with role-based capabilities rather than separate user types. A `user_roles` pivot gives a user `buyer` by default and `seller` once they activate selling.
- Session invalidation on password change / suspicious login — use Laravel's `invalidate()` on all sessions for that user.

**Implementation strategy:**
- Laravel Fortify for backend primitives (login, 2FA, email verification) + custom Livewire UI.
- Device tracking via a custom middleware that fingerprints (IP + User-Agent hash + cookie) on each auth action.
- Separate table `user_suspensions` — checked in a global middleware to short-circuit requests.

---

## 2.2 User Profiles

**Responsibility:** Public-facing profile (bio, skills, languages, education, certifications, portfolio), avatar, cover, URL slug (username), social links. Includes both display data and profile completeness scoring.

**Boundaries:** Owns `profiles`, `profile_skills`, `profile_languages`, `profile_education`, `profile_certifications`.

**Dependencies:** Identity (user_id), Shared Kernel.

**Risks:**
- Profile spoofing (copying another seller's portfolio).
- PII leaks (emails in bio, phone numbers).

**Challenges:**
- Username uniqueness + reservation (reserved usernames like `admin`, `support`, `api`).
- Slug changes — maintain old-slug redirects to preserve SEO.

**Implementation strategy:**
- Usernames are validated against a blocklist + profanity filter.
- Bio is sanitized server-side on save; regex for emails/phone numbers warns the user and flags the profile for review.
- Images uploaded through the File Upload module; profile stores references, not blobs.

---

## 2.3 Seller Workspace

**Responsibility:** The seller's private operational surface — gig manager, order inbox, earnings, analytics, availability settings, vacation mode, seller-level status panel.

**Boundaries:** Does NOT own gigs or orders — it is a composition layer. Exposes Livewire components + controllers for seller-facing pages.

**Dependencies:** Gigs, Orders, Wallet, Reviews, Messaging.

**Risks:** Becoming a god-module. Strict rule: no business logic here. Only UI composition and read-side queries.

**Challenges:**
- Analytics dashboards — must not hit write-side tables. Either a read replica or pre-aggregated daily snapshots.
- Vacation mode affects gig search visibility — publishes `SellerWentOnVacation` event, Gigs module listens and re-indexes.

**Implementation strategy:**
- Livewire 4 components with `#[Computed]` for cached reads.
- Dashboard widgets pull from a `seller_daily_stats` table refreshed nightly.

---

## 2.4 Buyer Workspace

**Responsibility:** Buyer's dashboard — active orders, message inbox, order history, saved gigs/sellers, browsing history, buyer requests.

**Boundaries:** Composition layer like Seller Workspace.

**Dependencies:** Orders, Messaging, Gigs (for saved/history), BuyerRequests.

**Risks:** Same as Seller Workspace — avoid business logic creep.

**Implementation strategy:** Livewire-heavy. URL query params (`#[Url]`) for filter persistence. Pagination via `wire:navigate`.

---

## 2.5 Gig Management

**Responsibility:** Gig CRUD, versioning, approval workflow, packaging (tiers), gig extras, gig FAQs, gig requirements questionnaire, gig gallery, gig status (draft/pending/active/paused/rejected/deleted).

**Boundaries:** Owns `gigs`, `gig_packages`, `gig_extras`, `gig_faqs`, `gig_requirements`, `gig_media`, `gig_tags`, `gig_metadata_values`, `categories`, `subcategories`.

**Dependencies:** Identity (seller_id), File Upload, Search (publishes events consumed by Search), Shared Kernel.

**Risks:**
- Gig edits while orders are in flight — orders must snapshot the gig state at order time (see Orders module).
- Massive gigs table — partitioning or careful indexing on `(status, category_id, seller_id)`.
- Metadata schema per category varies — do not over-engineer with EAV. Use a JSON column + per-category validation rules in PHP.

**Challenges:**
- Multi-language gigs — one gig has one primary language; translations are separate gig entities or a `gig_translations` table.
- Image requirements (aspect ratios, file size) — validated at upload, re-validated on approval.

**Implementation strategy:**
- `Gig` has states managed by an explicit state machine (spatie/laravel-model-states).
- Editing a live gig creates a `gig_versions` row; published version is the current row. Rollback = promote a version.
- Events: `GigCreated`, `GigSubmittedForReview`, `GigApproved`, `GigRejected`, `GigPaused`, `GigResumed`, `GigDeleted`.

---

## 2.6 Orders Management

**Responsibility:** The heart of the platform. Order creation, state machine, deadline management, delivery, revisions, completion, cancellation, order events (audit trail).

**Boundaries:** Owns `orders`, `order_items` (packages + extras), `order_requirements_responses`, `order_deliveries`, `order_revisions`, `order_events`, `order_milestones`.

**Dependencies:** Gigs (reads at order creation, snapshots), Identity (buyer, seller), Payments (publishes events), Messaging (injects system messages), Shared Kernel.

**Risks:**
- Invalid state transitions — any bug here costs money and trust. MUST be enforced in a single state machine class.
- Deadline calculation bugs (timezone, pauses, extensions) — all timestamps UTC, all business logic in a single `OrderDeadlineService`.
- Race conditions on concurrent actions (seller delivers while buyer cancels) — pessimistic locks on order row during state transitions.

**Challenges:**
- Order reference IDs — human-readable (`#FO2026-0001234`), collision-free under concurrency. Use a dedicated sequence or a ULID-derived format.
- Auto-complete timer (3 days after delivery) — scheduled via a delayed queue job + a nightly sweeper job as safety net.
- Milestones — child aggregate with their own state machine that rolls up to parent.

**Implementation strategy:**
- Single source of truth: `app/Modules/Orders/Application/Services/OrderStateMachine.php`. Every state change flows through one method: `$machine->transition($order, $event, $actor)`.
- Transitions emit events. Listeners are async (queued) except for the one that writes the `order_events` audit row, which is synchronous and inside the same DB transaction.
- Every action (deliver, revise, complete) is an Action class taking a command DTO and returning a DTO. Idempotency key required on every action.

See **Part 4** for the full state map.

---

## 2.7 Messaging System

**Responsibility:** Threads, messages, attachments, read receipts, typing indicators, online presence, system messages, custom offers as message types, content moderation (contact-info scanning).

**Boundaries:** Owns `threads`, `thread_participants`, `messages`, `message_attachments`, `message_flags`.

**Dependencies:** Identity, File Upload, Orders (for system-message injection), Reverb for real-time, Shared Kernel.

**Risks:**
- Hot-table problem — messages grow huge. Partition by month or archive cold threads.
- Abuse: off-platform contact info. Must scan every message server-side before persisting.
- WebSocket scaling — Reverb is fine up to mid-scale. Plan for horizontal scaling with Redis pub/sub.

**Challenges:**
- Ordering guarantees — use a monotonic `sequence_id` per thread, not created_at, to avoid millisecond collisions.
- Delivery vs read semantics — `delivered_at` when pushed to client, `read_at` when viewport-confirmed.

**Implementation strategy:**
- Messages are append-only. Edits create a new revision row; the UI shows "edited".
- Content scanning is a pipeline: regex pre-filter (emails, phones, common off-platform keywords) → async ML check (optional, v2) → flag/warn/block.
- Custom offers are `messages` with `type = 'custom_offer'` and a polymorphic `offer_payload` JSON. Accepting dispatches a command to Orders module.
- Real-time via Laravel Reverb + Echo on the client. Typing indicator is a Reverb whisper event, not persisted.

---

## 2.8 Notifications System

**Responsibility:** Multi-channel delivery (in-app, email, push, SMS), user preferences, notification history, digest batching.

**Boundaries:** Owns `notifications` (in-app), `notification_preferences`, `notification_digests`.

**Dependencies:** All other modules (as event consumers), Identity.

**Risks:**
- Notification storms — bulk events can blast thousands of emails. Rate-limit + batch.
- Email deliverability — SPF, DKIM, DMARC, dedicated IP warming. Use a reputable ESP (SES, Postmark).
- Preference drift — a user unsubscribes from email but the code keeps sending. Check preferences in the listener, not before dispatching the event.

**Challenges:**
- Digest logic — e.g., "you have 3 new messages" email sent hourly if the user didn't read prior push.
- Real-time in-app updates — Reverb channel per user.

**Implementation strategy:**
- Laravel's built-in Notification system extended. Each notification class declares channels it supports.
- A central `NotificationDispatcher` service reads user preferences + quiet hours + digest rules and decides what actually goes out.
- All email sending is queued.

---

## 2.9 Reviews System

**Responsibility:** Buyer reviews of sellers, seller responses, multi-axis ratings, review aggregation, review integrity checks.

**Boundaries:** Owns `reviews`, `review_responses`, `review_flags`, `seller_rating_aggregates` (denormalized read model).

**Dependencies:** Orders (reviews are post-completion), Identity.

**Risks:**
- Review manipulation — sockpuppets, swaps.
- Aggregation drift — the denormalized average drifts from the reality. Recompute nightly as a safety net.

**Challenges:**
- Editability window — 10 days post-submission, after which reviews are locked.
- Response window — seller has N days to respond.

**Implementation strategy:**
- A review can only be created if the linked order is `completed` and the review period (e.g., 30 days) hasn't elapsed.
- On review create, update `seller_rating_aggregates` inside the same DB transaction (not async — we need instant consistency on seller profile pages).
- Integrity: a fraud score per review computed async (buyer age, IP match to seller's past IPs, payment history, device uniqueness). Scores above a threshold flag the review for admin review (emit event to Admin app).

---

## 2.10 Checkout & Payments

**Responsibility:** Cart → checkout → payment authorization → capture → payout scheduling → refunds → chargebacks. Integrates with Stripe Connect.

**Boundaries:** Owns `checkouts`, `payments`, `payment_methods`, `stripe_customer_refs`, `stripe_connect_accounts`, `refunds`, `chargebacks`, `payment_events` (webhook audit), `ledger_entries`.

**Dependencies:** Orders (fires on payment confirmed), Identity, Wallet, Shared Kernel.

**Risks:**
- Double capture — idempotency critical.
- Webhook replay / out-of-order delivery — Stripe sends the same webhook multiple times; store event IDs, reject duplicates.
- Currency mismatch — platform currency vs payout currency vs card currency. Pick one accounting currency (USD), store all ledger entries in it + FX rate snapshot.
- Chargebacks — must freeze related order and seller payouts automatically.

**Challenges:**
- Stripe Connect destination charges vs separate charges + transfers — we use **separate charges and transfers** so we can split, delay, and reverse independently. Platform receives the full charge; transfers to seller happen on completion + clearance.
- PCI scope — never touch raw card data. Stripe Elements / PaymentElement on the frontend.
- Ledger — every cent movement is two entries (double-entry). Balances are derived from ledger, never stored as a mutable column.

**Implementation strategy:**
- Dedicated `LedgerService` — the only place ledger entries are written. Every entry has: account, amount (signed), currency, direction, order_id, payment_id, reason code, idempotency_key, timestamp.
- Webhooks: `POST /webhooks/stripe` → verify signature → persist raw event → enqueue processing job → ack 200 immediately.
- All money is stored in minor units (cents) as `BIGINT`. No floats anywhere.

See **Part 3** for ledger design.

---

## 2.11 Wallet / Revenue Tracking

**Responsibility:** Seller earnings view, buyer credit balance, withdrawal requests, payout history.

**Boundaries:** Owns `wallet_accounts` (one per user, per currency), `withdrawal_requests`, `payouts`.

**Dependencies:** Payments (all actual money movement), Identity.

**Risks:**
- Wallet-balance-drift from ledger — balances MUST be derived from ledger sums, not stored. If we cache balance, invalidate on every ledger write.
- Withdrawal race — user clicks "withdraw" twice, two payouts. Idempotency + pessimistic lock on wallet row.

**Implementation strategy:**
- Wallet balance is a view (SQL VIEW) or a cached computation: `SUM(ledger_entries.amount) WHERE account_id = ?`.
- Withdrawals have their own state machine: `requested → processing → paid → failed`. Failed payouts return the funds to the wallet via a reverse ledger entry.

---

## 2.12 Refunds / Cancellations

**Responsibility:** Process refunds and cancellations triggered by order state changes, dispute resolutions, or admin action.

**Boundaries:** This is mostly a coordinator. It listens for events from Orders and Disputes, instructs Payments to refund, updates ledger.

**Dependencies:** Orders, Payments, Disputes, Wallet, Notifications.

**Risks:**
- Partial refund math — must handle gig extras proportionally, platform fees handled correctly (Fiverr keeps the fee on cancelled orders in some cases? we charge no fee on full cancellation pre-delivery).
- Refund to original method vs wallet — both supported, different Stripe calls.

**Implementation strategy:**
- `RefundService::issueRefund(Order $order, Money $amount, RefundDestination $destination, RefundReason $reason, string $idempotencyKey)`.
- Creates a `refunds` row in `pending`, fires Stripe refund, updates ledger, updates order, publishes `RefundCompleted` or `RefundFailed`.

---

## 2.13 Dispute Center

**Responsibility:** Structured conflict resolution between buyer and seller with counter-offers, timeouts, and escalation.

**Boundaries:** Owns `disputes`, `dispute_proposals`, `dispute_events`.

**Dependencies:** Orders, Messaging (dispute creates a system message), Notifications.

**Risks:**
- Timer bugs — 48h response windows. Same UTC + pause rules as order deadlines.
- Escalation loops — bounded number of counter-offers (3 max) before forced escalation to admin.

**Implementation strategy:**
- Dispute is tied to one order. Only one active dispute per order.
- Actions: `openDispute`, `proposeCounter`, `accept`, `reject`, `escalate`, `withdraw`.
- On accept, dispatches commands to Orders/Refunds to enact the proposed resolution.
- Escalation publishes event to Admin app (out of scope here) — we just set `status = escalated` and notify.

---

## 2.14 Search System

**Responsibility:** Indexing gigs into Elasticsearch, serving search queries, autocomplete, faceted filters, re-ranking.

**Boundaries:** Owns the Elasticsearch mapping definitions, index strategy, and search query builders. Exposes a `GigSearchContract` interface.

**Dependencies:** Gigs (consumer of events), Sellers (metric changes), Reviews (rating changes), Shared Kernel.

**Risks:**
- Index drift — ES out of sync with MySQL. Mitigations: (a) all writes go through a `GigIndexer` that's called by listeners; (b) nightly full reindex job; (c) on-read verification for small result sets (compare first page to DB — for suspected drift only).
- Query cost — `function_score` can get expensive. Profile + cache common queries.
- Relevance quality — requires iteration, A/B testing, feedback loops.

**Challenges:**
- Reindexing during schema changes — blue/green index strategy (alias `gigs_live` pointing to `gigs_v5`, build `gigs_v6` in background, flip alias).
- Personalization — do it at re-rank time, not in the ES query.

**Implementation strategy:**
- ES client: `elastic/elasticsearch-php`.
- Index name alias: `gigs` → `gigs_YYYYMMDD`.
- Indexer is an async queued listener on `GigApproved`, `GigUpdated`, `SellerMetricsUpdated`, `ReviewAggregateChanged`.
- Dedicated `search` queue with Horizon, separate from critical-path queues.

---

## 2.15 Coupons & Promotions

**Responsibility:** Coupon codes, referral credits, first-order discounts.

**Boundaries:** Owns `coupons`, `coupon_usages`, `referrals`.

**Dependencies:** Checkout (validation at checkout time), Identity.

**Risks:**
- Stacking bugs leading to negative order totals. Validate at checkout + re-validate at capture.
- Code brute-forcing — rate limit code application attempts.

**Implementation strategy:**
- Single `CouponValidationService` used at checkout, authoritative.
- Pessimistic lock on `coupons` row when checking `used_count < max_uses` to prevent over-redemption.

---

## 2.16 Trust & Safety

**Responsibility:** Cross-cutting anti-abuse — content moderation signals, fraud scoring, device fingerprinting, seller-level computation, warnings, restrictions.

**Boundaries:** Listens to events from across the platform. Owns `user_warnings`, `user_restrictions`, `fraud_signals`, `device_fingerprints`, `seller_level_snapshots`.

**Dependencies:** Effectively everything (event consumer).

**Risks:**
- Becoming a god-module — it has tentacles everywhere. Discipline: only reads via events, writes only to its own tables (except for restriction enforcement which is checked via middleware from its tables).

**Implementation strategy:**
- Listeners subscribe to events like `MessageCreated` (scan), `OrderCompleted` (recompute seller metrics), `ReviewCreated` (fraud score), etc.
- Seller level recomputation is a scheduled nightly command that reads order/review aggregates for a rolling 60-day window.

---

## 2.17 Activity Logs

**Responsibility:** Cross-cutting audit of significant actions.

**Boundaries:** Write-only from the rest of the system. Owns `activity_log`.

**Dependencies:** None (pure observer).

**Implementation strategy:**
- `spatie/laravel-activitylog` or a custom append-only table.
- Logged via listeners on domain events.
- Partitioned by month; old logs archived to S3.

---

## 2.18 File Upload System

**Responsibility:** Direct-to-S3 uploads, signed URL generation, virus scanning, image processing (resize, WebP, thumbnail), video processing, metadata extraction.

**Boundaries:** Owns `uploads`, `upload_variants`. Exposes an interface so other modules reference `upload_id`, not raw paths.

**Dependencies:** Shared Kernel.

**Risks:**
- Unrestricted uploads (size, type) — validate on client AND server.
- Malware — ClamAV or cloud scanner (e.g., AWS GuardDuty for S3).
- Orphan files — scheduled cleanup of uploads not referenced by any module within 24h.

**Implementation strategy:**
- Uploads go direct to S3 with pre-signed POST URLs. The server never touches the bytes on upload.
- On successful upload, client calls our API `/uploads/confirm` → we verify the object exists + scan + enqueue processing.
- Variants (thumbnail, webp, medium) generated in queued jobs, stored with upload reference.

---

## 2.19 Real-time Updates

**Responsibility:** WebSocket infrastructure (Laravel Reverb), channel authorization, broadcasting domain events.

**Boundaries:** Infrastructure. Broadcast contracts are defined by the modules that emit them; this module provides the transport.

**Dependencies:** Identity (for channel auth), all modules (as broadcasters).

**Risks:**
- Auth bypass — channel authorization must always check ownership.
- Reverb scaling — single-node limits. For horizontal scale, run multiple Reverb nodes behind Redis pub/sub.

**Implementation strategy:**
- Private channels: `user.{userId}`, `thread.{threadId}`, `order.{orderId}`.
- Every broadcastable event implements `ShouldBroadcast` and goes through the `broadcast` queue.
- Reverb runs under its own supervisor process.

---

## 2.20 Shared Kernel

**Responsibility:** Truly cross-cutting, stable abstractions: `Money`, `Currency`, `Uuid`/`Ulid`, `IdempotencyKey`, base exceptions, common middleware, utility helpers, enums.

**Boundaries:** Minimal. The rule: if it's not used by ≥3 modules and guaranteed stable, it doesn't belong here.

**Risks:** Bloat. Every PR adding to Shared Kernel requires two senior reviewers.

---

# PART 3 — DATABASE & DOMAIN STRATEGY

## 3.1 Core entities

The core aggregates (each is a consistency boundary — a single transaction may only span one aggregate):

- **User** — root of Identity. Children: Sessions, Devices.
- **Profile** — one-to-one with User.
- **Gig** — root of Gigs. Children: Packages, Extras, FAQs, RequirementQuestions, Media, Tags.
- **Order** — root of Orders. Children: Items, RequirementResponses, Deliveries, Revisions, Events, Milestones.
- **Thread** — root of Messaging. Children: Messages, Attachments, Participants.
- **Payment** — root of Payments. Children: Refunds, Chargebacks, WebhookEvents.
- **WalletAccount** — root of Wallet. Children: WithdrawalRequests, Payouts.
- **Dispute** — root of Disputes. Children: Proposals, Events.
- **Review** — root of Reviews. Children: Response.
- **LedgerEntry** — not an aggregate in the DDD sense; it's an append-only fact.

## 3.2 Value Objects

- **Money** — `amount: int` (minor units), `currency: string(3)`. Immutable. Arithmetic via methods (`add`, `subtract`, `percentage`). No floats.
- **OrderReference** — human-readable order ID wrapper with format validation.
- **IdempotencyKey** — opaque string, 64-char max, guaranteed unique per logical operation.
- **Email**, **PhoneNumber**, **Username** — validated wrappers.
- **DateRange**, **Duration** — for deadlines, pauses.
- **Slug** — URL-safe string with reserved-word checks.

These are PHP classes with `readonly` properties (PHP 8.3). They're not Eloquent casts unless we need them on model attributes (we will — `Money` becomes a cast).

## 3.3 Domain Services

Cross-aggregate business logic that doesn't fit naturally in one aggregate:

- **OrderStateMachine** — validates transitions, emits events.
- **OrderDeadlineService** — computes effective deadlines given pauses, revisions, extensions.
- **PricingCalculator** — given gig + package + extras + coupons, returns an immutable `PriceQuote`.
- **LedgerService** — only writer of ledger entries.
- **SellerLevelCalculator** — reads metrics, outputs a level decision.
- **FraudScorer** — composes signals into a score.

## 3.4 Application Services (Actions)

Thin orchestrators triggered by controllers/Livewire. One class per use case:

```
CreateOrderAction
DeliverOrderAction
RequestRevisionAction
AcceptDeliveryAction
CancelOrderAction
RefundOrderAction
OpenDisputeAction
RespondToDisputeAction
PublishGigAction
WithdrawEarningsAction
```

Rule: Actions contain only orchestration — they call domain services, Eloquent, and dispatch events. They do not contain business rules.

## 3.5 Event-driven flows

We use **two kinds of events**:

1. **Domain events** (internal to a module): synchronous, used inside the module for cohesion.
2. **Integration events** (cross-module): asynchronous, queued, named with the past tense (`OrderCompleted`, not `CompleteOrder`).

Integration events are the ONLY way modules communicate asynchronously. Every integration event has:

- A unique event ID (ULID).
- `occurred_at` UTC.
- `aggregate_id` (e.g., order ID).
- `payload` (immutable data — NOT a reference to a mutable model; if listeners need current state, they re-fetch).
- `version` (for payload schema evolution).

**Inbox pattern** for critical listeners: the listener first records the event ID in an `event_inbox` table (unique constraint on event_id + listener_name). If the insert fails (duplicate), the event has already been processed — skip. This provides exactly-once processing on top of at-least-once delivery from queues.

**Outbox pattern** for events that must not be lost (e.g., payment confirmed → must publish `PaymentCompleted`): write the event to an `event_outbox` table in the same DB transaction as the state change, then a separate worker publishes from the outbox.

## 3.6 Queue usage

Queues (via Horizon):

- `critical` — payment webhooks, order state transitions, ledger writes. Max 10 workers, auto-scale aggressive.
- `default` — most application jobs.
- `notifications` — emails, push, SMS. Lower priority.
- `search` — ES indexing. Separate so bad search backlog doesn't starve order processing.
- `media` — image/video processing. Long-running, separate workers.
- `broadcast` — Reverb broadcasting.
- `exports` — CSV exports, large reports.

Every job:

- Has a `tries` limit appropriate to idempotency (idempotent jobs can retry aggressively; non-idempotent need a single attempt + dead-letter).
- Has a `backoff` schedule (exponential, jittered).
- Implements `ShouldBeUnique` where applicable to prevent duplicate processing.
- Has a `retryUntil` for time-bounded retries.
- Writes a failure record to a dead-letter table on final failure.

## 3.7 Redis usage

- **Cache** — gig page rendered fragments, category trees, user permissions cache.
- **Session store** — in production, Redis-backed sessions.
- **Rate limiting** — Laravel's `RateLimiter` → Redis.
- **Locks** — `Cache::lock()` for distributed locks (e.g., on seller withdrawal).
- **Queue backend** — yes (via Horizon).
- **Reverb pub/sub** — for horizontal WebSocket scaling.
- **Short-lived data** — typing indicators, online presence (TTL 60s).

Do NOT use Redis as primary store for anything. Redis is ephemeral.

## 3.8 Search indexing strategy

- Index on event, not on save. Eloquent model events → domain event → queued indexer.
- Use aliases for blue/green reindexing.
- Document ID = gig ID.
- Heavy denormalization: seller fields (level, country, rating, response_rate) copied into gig docs. When seller metrics change, all their gigs reindex (N is small — a seller has tens of gigs, not thousands).
- Delete-by-query forbidden in production; use explicit deletes per document.

## 3.9 Payment transaction consistency

**Rules**:

1. Every money-touching operation has an idempotency key.
2. Every money-touching operation writes to the ledger in the same DB transaction as the state change.
3. External calls (Stripe) happen OUTSIDE the DB transaction. Pattern: write intent → commit → call Stripe → update with result in new transaction.
4. Failures during step 3 leave a consistent `pending` state that a reconciliation job resolves.
5. Ledger is the source of truth; balances are derived.

**Ledger schema** (simplified):

```
ledger_entries
  id                ULID, PK
  account_id        FK to ledger_accounts
  amount            BIGINT (minor units, signed)
  currency          CHAR(3)
  direction         ENUM('debit', 'credit')
  order_id          FK nullable
  payment_id        FK nullable
  withdrawal_id     FK nullable
  reason_code       VARCHAR (SEED-enum from config)
  idempotency_key   VARCHAR UNIQUE
  metadata          JSON
  created_at        TIMESTAMP
```

Accounts: `platform_revenue`, `platform_fees`, `seller_escrow_{id}`, `seller_available_{id}`, `buyer_credit_{id}`, `stripe_clearing`, `refunds_pending`. Every financial event is a balanced pair (or more) of entries.

## 3.10 Idempotency strategy

- Every write API endpoint accepts `Idempotency-Key` header. Persisted in `idempotency_keys` table with request hash + response hash. Repeat requests return the stored response.
- Every job that mutates money or order state is keyed by a deterministic key derived from inputs.
- Stripe webhooks: Stripe's event ID is the idempotency key.

## 3.11 Concurrency strategy

- **Pessimistic locks** (`SELECT ... FOR UPDATE`) on: order state transitions, wallet balance reads prior to write, coupon redemption.
- **Optimistic locks** (version column) on: profile updates, gig edits (to warn on concurrent edits).
- **Distributed locks** (Redis via `Cache::lock`) on: scheduled recomputations to prevent duplicate runs across nodes, withdrawal initiation per-user.

## 3.12 Failure recovery strategy

- **Reconciliation jobs** (nightly):
  - Payments vs ledger — alert on any unbalanced order.
  - ES vs MySQL — compare doc counts and sample gigs; reindex discrepancies.
  - Stripe balance vs platform ledger — must match to the cent.
  - Orphan uploads — delete after 24h if unreferenced.
  - Stuck orders — in a state longer than sane (e.g., `pending_requirements` for >7 days) → escalate to admin.
- **Dead-letter queue** — failed jobs surfaced in Horizon + daily Slack digest.
- **Webhook replay** — admin can replay any Stripe webhook from the raw event log.

## 3.13 Preventing common failure patterns

- **Duplicated services**: enforced by module boundaries + CI check (`PricingCalculator` only in `Checkout` module).
- **Conflicting business logic**: one canonical domain service per concept. No "helper" classes.
- **Tight coupling**: modules expose only `Contracts/`. CI fails on cross-module internal imports.
- **Scalability foot-guns**: no N+1 allowed in code review (use `spatie/laravel-eloquent-strictness` or enable `Model::preventLazyLoading()` in non-prod).
- **Transactional inconsistencies**: the ledger rule + outbox pattern for cross-aggregate consistency.

---

# PART 4 — ORDER LIFECYCLE MAP

This is the single most important section of the document. The order state machine is the contract between the platform and its users. Any bug here results in money loss or trust loss. Implement it in **one class** (`OrderStateMachine`), test it exhaustively, and never bypass it.

## 4.1 Order states (enum)

```
DRAFT                   // Pre-creation (checkout cart). Not yet persisted as order.
PAYMENT_PENDING         // Order row exists, payment not yet captured.
PAYMENT_FAILED          // Terminal for failed checkout. Buyer can retry (new order).
PENDING_REQUIREMENTS    // Paid, awaiting buyer to submit requirements answers.
IN_PROGRESS             // Requirements received, seller's delivery clock ticking.
DELIVERED               // Seller submitted delivery. Auto-complete clock ticking.
REVISION_REQUESTED      // Buyer requested revision. Delivery clock resumes.
COMPLETED               // Buyer accepted OR auto-completed after 3 days.
CANCELLED               // Mutually or unilaterally cancelled. Refund executed or in-flight.
DISPUTED                // Active dispute in Resolution Center (order frozen).
REFUNDED                // Terminal after full refund (may follow CANCELLED or DISPUTED).
LATE                    // Virtual flag, not a terminal state — combined with IN_PROGRESS/DELIVERED.
```

`LATE` is not a state — it's a computed flag (`is_late = deadline < now() AND status IN (IN_PROGRESS, REVISION_REQUESTED)`). It affects UI, metrics, and enables buyer actions (cancel with penalty to seller).

## 4.2 Transition map

Legend: `[ACTOR]` who triggers, `{condition}` precondition.

```
DRAFT → PAYMENT_PENDING
  [BUYER] submits checkout
  {payment method valid, gig active, no buyer restrictions}

PAYMENT_PENDING → PENDING_REQUIREMENTS
  [SYSTEM] on Stripe webhook: payment_intent.succeeded
  {idempotent, payment captured}

PAYMENT_PENDING → PAYMENT_FAILED
  [SYSTEM] on Stripe webhook: payment_intent.payment_failed
  {terminal for this order}

PENDING_REQUIREMENTS → IN_PROGRESS
  [BUYER] submits requirements
  {all required questions answered}
  side-effect: start delivery timer (package.delivery_days + extras.additional_days)

PENDING_REQUIREMENTS → CANCELLED
  [BUYER/SELLER/SYSTEM] cancel
  {if SYSTEM: no requirements submitted within 7 days → auto-cancel}
  side-effect: full refund

IN_PROGRESS → DELIVERED
  [SELLER] submits delivery
  {at least one deliverable file OR delivery message}
  side-effect: pause delivery timer, start auto-complete timer (3 days)

IN_PROGRESS → CANCELLED
  [BUYER] cancels with late-order penalty
  {is_late = true} OR {mutual cancel accepted via dispute}
  side-effect: full refund; seller metrics impacted

IN_PROGRESS → DISPUTED
  [BUYER/SELLER] opens dispute
  side-effect: pause delivery timer

DELIVERED → COMPLETED
  [BUYER] accepts delivery
  OR [SYSTEM] auto-complete after 3 days
  side-effect: stop auto-complete timer; seller funds move from escrow to clearing (14-day hold)

DELIVERED → REVISION_REQUESTED
  [BUYER] requests revision
  {revisions_used < package.revisions + extras.extra_revisions}
  side-effect: pause auto-complete timer, resume delivery timer (with configured extension)

DELIVERED → DISPUTED
  [BUYER/SELLER] opens dispute
  side-effect: pause auto-complete timer

REVISION_REQUESTED → DELIVERED
  [SELLER] redelivers
  side-effect: same as IN_PROGRESS → DELIVERED

REVISION_REQUESTED → CANCELLED
  [BUYER/SELLER/SYSTEM] mutual cancel via dispute

REVISION_REQUESTED → DISPUTED
  [either party] opens dispute

DISPUTED → IN_PROGRESS
  [SYSTEM] dispute resolved with "continue work" proposal

DISPUTED → COMPLETED
  [SYSTEM] dispute resolved with "accept delivery" proposal

DISPUTED → CANCELLED
  [SYSTEM] dispute resolved with "cancel" proposal

DISPUTED → (stays DISPUTED, escalated flag set)
  [SYSTEM] after 48h no response → escalation

CANCELLED → REFUNDED
  [SYSTEM] refund completes successfully
  {payment captured AND refund confirmed by Stripe OR wallet credit issued}

CANCELLED → CANCELLED  (retry-safe)
  [SYSTEM] refund retry after transient failure

COMPLETED → (terminal, allows review submission for 30 days)
REFUNDED → (terminal)
PAYMENT_FAILED → (terminal)
```

## 4.3 Invalid transitions (must throw)

Every transition not explicitly listed above MUST throw `InvalidOrderTransition`. Examples the state machine must reject:

- `COMPLETED → IN_PROGRESS` (re-opening a completed order — never, use a new order)
- `REFUNDED → *` (refunded is terminal)
- `DRAFT → IN_PROGRESS` (skipping payment)
- `DELIVERED → IN_PROGRESS` (without a revision request)
- `CANCELLED → COMPLETED` (zombie order)
- Any transition while `is_frozen = true` (frozen by fraud hold) except admin-forced transitions

## 4.4 Timers and pauses

Each order tracks:

- `delivery_deadline_at` — computed at transition into IN_PROGRESS; paused on DELIVERED and DISPUTED.
- `delivery_remaining_seconds` — snapshot on pause.
- `auto_complete_deadline_at` — set on DELIVERED; paused on REVISION_REQUESTED and DISPUTED.
- `auto_complete_remaining_seconds` — snapshot on pause.

On resume, deadline = `now() + remaining_seconds`.

All timer logic in `OrderDeadlineService`. Scheduled sweeper job runs every 5 minutes:

- Finds orders where `auto_complete_deadline_at <= now()` AND `status = DELIVERED` → transition to COMPLETED.
- Finds orders where `delivery_deadline_at <= now()` AND `status IN (IN_PROGRESS, REVISION_REQUESTED)` → mark `is_late = true`, fire `OrderBecameLate` event.

Sweeper must be idempotent (it runs every 5 minutes; transitions must be safe on retry).

## 4.5 Edge cases

- **Buyer requests refund on a COMPLETED order within the review window.** Not allowed — only path is a TOS-violation report to admin. We do NOT allow free unwinds post-completion.
- **Seller delivers on time, buyer goes dark, auto-completes.** Expected path. Seller paid. Buyer has 10 days to leave a review.
- **Seller delivers after deadline (late delivery).** Delivery still accepted, but `delivered_late = true` flag. Buyer can cancel with the late penalty option up to the moment of delivery. Post-delivery, late flag affects seller metrics but buyer flow continues normally.
- **Buyer requests revision after all revisions used.** UI prevents it, but server double-checks. If they have a "goodwill" channel via messaging, seller can manually trigger a free revision via a specific action.
- **Milestone order partially delivered.** Each milestone has its own state. Parent order is COMPLETED only when all milestones are COMPLETED. Refunds/cancellations can be per-milestone.
- **Seller account suspended mid-order.** All active orders are paused (frozen). Admin intervention required. Buyer can request refund via admin channel. The state machine must handle `is_frozen` and reject most transitions until thawed.
- **Chargeback received on a COMPLETED order.** Immediate state flip to `DISPUTED` with `chargeback_in_progress` flag. Seller earnings for this order clawed back from clearing/available balance. If balance insufficient, negative wallet balance → seller cannot withdraw until cleared.
- **Concurrent actions**: Buyer clicks "Accept Delivery" at the same moment as auto-complete fires. Pessimistic lock on order row — whichever acquires first wins, the other sees `InvalidOrderTransition` (already COMPLETED) and the client shows success regardless.
- **Buyer deletes account mid-order.** Account deletion requires no active orders. Block deletion; show list of orders that must complete first.
- **Seller deletes gig mid-order.** Gigs are soft-deleted and cannot be deleted while active orders exist. Status goes to `archived` instead; existing orders continue with the snapshot data captured at order time.

## 4.6 Timeout cases

| Timeout | Trigger | Action |
|---|---|---|
| Requirements not submitted in 7 days | `PENDING_REQUIREMENTS` > 7d | Auto-cancel, full refund |
| Auto-complete | `DELIVERED` + 3d elapsed (no pause active) | Transition to `COMPLETED` |
| Dispute response | Counter-proposal not acted on in 48h | Escalate to admin (flag only; state stays DISPUTED) |
| Delivery deadline passed | `IN_PROGRESS` past deadline | Flag `is_late`, notify buyer, enable late-cancel |
| Review window | 30 days post-COMPLETED | Lock review submission |
| Review edit window | 10 days post-submission | Lock review edits |
| Seller response window | 24h after seller posts response | Lock response edits |

## 4.7 Fraud prevention hooks (on state transitions)

- **On PAYMENT_PENDING → PENDING_REQUIREMENTS**: check buyer fraud score. If > threshold, flag order for manual review before releasing to seller (`is_frozen = true`, admin reviews within 1h SLA).
- **On IN_PROGRESS → DELIVERED**: scan delivered files for known-malicious hashes. Block if match.
- **On DELIVERED → COMPLETED**: if this is the buyer's first order AND completion was <5 minutes after delivery AND high value → flag for review. Legitimate fast acceptance exists, but combined with first-order + high-value, it correlates with shill reviews.
- **On COMPLETED**: recompute seller metrics (async).
- **On CANCELLED (by buyer)**: increment buyer's cancellation rate counter. Above threshold → reduce buyer trust score (affects future order approvals and refund eligibility).

## 4.8 Auditability

Every transition writes a row to `order_events` in the same transaction:

```
order_events
  id              ULID
  order_id        FK
  from_status     ENUM
  to_status       ENUM
  actor_type      ENUM('buyer', 'seller', 'system', 'admin')
  actor_id        nullable
  reason          VARCHAR (reason code)
  metadata        JSON (e.g., refund amount, extension days)
  idempotency_key VARCHAR UNIQUE
  created_at      TIMESTAMP
```

This table is append-only. Every customer support conversation starts by pulling this timeline.

---

# PART 5 — GITHUB ENGINEERING RULES

These rules are NOT negotiable. PRs that violate them are rejected by CI or by review. The goal is to prevent the slow, invisible erosion of quality that kills projects at month 18.

## 5.1 Repository layout

Single monorepo containing:

- `/` — the Laravel app (modular monolith).
- `/docs/` — architecture docs, ADRs, runbooks.
- `/infra/` — Terraform/Pulumi + deployment configs.
- `/.github/` — workflows, CODEOWNERS, issue and PR templates.

The Admin Panel application lives in a separate repository.

## 5.2 Branch naming

```
main                        // always deployable, protected
develop                     // integration branch, protected (optional — see 5.5 merge strategy)
feature/<ticket>-<slug>     // feature/ORD-142-order-state-machine
fix/<ticket>-<slug>
hotfix/<ticket>-<slug>      // branched from main, merged to main AND develop
chore/<ticket>-<slug>
refactor/<ticket>-<slug>
docs/<ticket>-<slug>
release/<version>           // release/1.4.0 — for cutting releases if not trunk-based
```

Branch names are kebab-case. Tickets are uppercase prefix + number matching the project management tool.

## 5.3 Commit messages

Conventional Commits format, enforced by commitlint:

```
<type>(<scope>): <subject>

<body>

<footer>
```

Types: `feat`, `fix`, `refactor`, `perf`, `test`, `docs`, `chore`, `ci`, `revert`.

Scope is the module name (`orders`, `gigs`, `payments`, `messaging`, ...).

Example:
```
feat(orders): add auto-complete sweeper job

Scheduled every 5 minutes to transition DELIVERED orders past
their auto-complete deadline to COMPLETED. Idempotent — safe
on retry, uses pessimistic lock per order.

Closes ORD-142
```

## 5.4 Pull requests

**PR template (`.github/pull_request_template.md`)** must include:

- Ticket link
- Summary (what + why)
- Scope (which modules touched)
- Breaking changes (yes/no + migration notes)
- Tests added / modified
- Manual test steps
- Screenshots/recordings for UI
- Checklist: migrations, queue jobs, env vars, feature flags, docs updated, ADR written (if architectural)

**PR size limit:** soft 400 lines of diff, hard 800. Larger PRs are split or justified in the description.

**PR title:** Conventional Commits format, same as commit subject.

**Draft PRs** are required for early feedback. Ready-for-review is a separate state.

## 5.5 Merge strategy

**Trunk-based with squash merges to `main`** is the default. No `develop` unless the team size justifies it (>12 engineers).

- All PRs merge to `main` via squash.
- PR branches deleted on merge.
- `main` is always deployable; every merge to `main` triggers CI and a deploy to staging.

Rebase merges allowed for release/hotfix branches.

Merge commits forbidden in regular flow.

## 5.6 Review process

- **Minimum 1 approver** for normal PRs, **2 approvers** for:
  - Migrations
  - Money-touching code (Payments, Wallet, Ledger)
  - Order state machine
  - Shared Kernel changes
  - Security-sensitive code (auth, permissions, file uploads)
- **CODEOWNERS** file enforces review requirements per module. Every module has a designated owner (senior engineer).
- Reviewers check: correctness, tests, naming, module boundary compliance, security checklist.
- Author self-reviews before requesting review (walks through the diff with commentary).
- SLA: first review within 4 business hours for normal PRs, 1h for hotfixes.

## 5.7 CI requirements

Every PR runs:

1. **Lint** — Laravel Pint (PSR-12), Prettier for Blade/JS.
2. **Static analysis** — PHPStan level 8.
3. **Architecture check** — Deptrac (module boundaries) + custom rules.
4. **Security** — `composer audit`, Snyk/Dependabot.
5. **Tests** — Pest unit + feature. Minimum 80% coverage on new code; global coverage must not drop.
6. **Migration dry-run** — verify migrations run against a fresh DB + run down().
7. **OpenAPI / Postman contract check** — if API changed, verify contract tests still pass.
8. **Build check** — assets compile, config cache builds.

PRs cannot merge if any CI step fails. No exceptions.

## 5.8 Code ownership

CODEOWNERS enforces review assignment:

```
# Default
* @team/engineers

# Core modules
app/Modules/Orders/          @team/leads @alice
app/Modules/Payments/        @team/leads @bob
app/Modules/Identity/        @team/leads
app/Modules/Messaging/       @charlie

# Shared Kernel — everyone must approve
app/SharedKernel/            @team/leads @team/seniors

# Infra
infra/                       @team/leads @devops
.github/workflows/           @team/leads @devops
database/migrations/         @team/leads

# Architecture
docs/adr/                    @team/leads
```

## 5.9 Architecture enforcement

**Deptrac** configuration forbids:

- Cross-module imports except via `Contracts/` or `SharedKernel/`.
- `App\Http` depending on module internals.
- Module tests importing from other modules.

**Custom PHPStan rules** (via `larastan` + custom extensions):

- No `DB::raw()` without binding.
- No `env()` calls outside of config files.
- All Eloquent queries must include explicit `select()` or the model must define `$visible`.
- No `Model::preventLazyLoading(false)` in committed code.
- Actions must have a single public method (`execute`, `handle`, `__invoke`).

## 5.10 Forbidden practices

Hard bans — PRs containing these fail CI or are rejected in review:

- `dd()`, `dump()`, `var_dump()`, `print_r()`, `die()` in non-test code.
- Nested transactions beyond one level (use DB savepoints explicitly or redesign).
- Business logic in Blade templates or controller methods beyond ~10 lines.
- Direct database writes in listeners that should go through Actions.
- Float arithmetic on money.
- Raw SQL without query bindings.
- `Model::unguard()` in application code.
- `eval()`, unsafe deserialization of user input.
- Storing secrets in code or non-encrypted config.
- `Cache::forever()` without explicit justification.
- `composer update` in PRs (only `composer require <package>` + commit updated lock).
- Use of Livewire `$this->emit()` (removed in v4).
- Massive migration files (>1 table per migration where possible; 1 logical change per migration always).
- Mixing application code with Filament code (Filament lives in the separate admin repo).

## 5.11 Deployment workflow

**Environments:** `local` → `staging` → `production`.

- **Staging**: auto-deployed on every merge to `main`. Runs a superset of prod data (anonymized). Smoke tests run post-deploy.
- **Production**: manual approval gate after staging green for >15 minutes. Zero-downtime deploy: migrations run in a separate step before code deploy, new code is backward-compatible with the previous schema for one release cycle.
- **Blue/green** at the infrastructure level. Health-check based switchover.
- **Database migrations** are additive for at least one release. Destructive migrations (drop column, drop table) happen in a follow-up PR after the new code has been stable for a week.
- **Feature flags** (e.g., Laravel Pennant) for risky features. Default off, enabled in stages.

## 5.12 Release workflow

Semantic versioning: `MAJOR.MINOR.PATCH`.

- MINOR releases weekly on Wednesday.
- PATCH releases as needed.
- MAJOR releases with a migration plan and announcement.

Release checklist (`/docs/release-checklist.md`):

- Changelog updated.
- Migrations reviewed.
- Feature flag plan.
- Rollback plan documented.
- Stakeholders informed.
- Support team briefed on user-facing changes.

Tagged releases in GitHub with generated changelog.

## 5.13 Rollback strategy

- **Code rollback**: re-deploy previous tag via CI. Must complete in <5 minutes.
- **Migration rollback**: `down()` methods MUST exist and be tested. For destructive migrations that can't be rolled back, the plan is to roll forward with a compensating migration (and the prior release cycle should cover this).
- **Data rollback**: point-in-time restore from managed DB backups (RDS, etc.). Documented RPO: 5 minutes. RTO: 30 minutes.
- **Feature flag rollback**: flip flag off — no code deploy required.

## 5.14 Hotfix workflow

- Branch from `main`: `hotfix/<ticket>-<slug>`.
- PR with expedited review (1 senior approver, can merge without normal SLA).
- Deploy to staging + smoke test (minimum 15 minutes if feasible; 5 minutes if critical).
- Deploy to production.
- Post-mortem within 48h for any hotfix caused by production incident.

## 5.15 Issue tracking standards

Tickets in Linear/Jira with required fields:

- Type (feature, bug, task, spike).
- Module scope.
- Priority (P0 outage → P4 nice-to-have).
- Acceptance criteria.
- Links to design/spec.
- Estimate (story points or time).

Bug reports require: repro steps, expected, actual, environment, screenshots.

## 5.16 Milestone strategy

Work is organized into **sprints** (2 weeks) within **quarterly milestones**. Each milestone has:

- A goal statement.
- Measurable outcomes.
- Scope boundary (what's IN, what's OUT).
- A ship date.

Spillover is tracked and the scope cut, not the date (fixed-time, variable-scope).

## 5.17 Documentation rules

Mandatory docs:

- **README.md** (root) — setup, key commands, architecture overview, links.
- **`/docs/architecture/`** — this document + module docs.
- **`/docs/adr/`** — Architecture Decision Records (see 5.18).
- **`/docs/runbooks/`** — on-call runbooks for critical systems (payments, messaging, search).
- **Per-module README** — responsibilities, contracts, events, key classes.
- **API docs** — OpenAPI spec, auto-generated where possible.

Docs are reviewed in PRs like code. Outdated docs are treated as bugs.

## 5.18 ADR rules

**Architecture Decision Records** — short markdown files in `docs/adr/` documenting significant decisions.

Format (`adr-0012-use-stripe-connect-separate-charges-and-transfers.md`):

```
# ADR 0012: Use Stripe Connect Separate Charges and Transfers

- Status: Accepted
- Date: 2026-01-15
- Deciders: @alice, @bob, @tech-leads

## Context
What problem are we solving? What are the constraints?

## Decision
What we decided.

## Consequences
- Positive: ...
- Negative: ...
- Neutral: ...

## Alternatives considered
1. Destination charges — rejected because ...
2. Direct charges — rejected because ...
```

An ADR is required for:

- Choosing or changing a core technology (queue, search, cache, payment provider).
- Introducing a new module or changing module boundaries.
- Changing a state machine.
- Changes to the ledger or money model.
- Changes to event contracts that are already in use.

ADRs are immutable once Accepted — superseded by later ADRs, not edited.

---

# PART 6 — DEVELOPMENT ROADMAP

Timeframes are rough; your team size will dictate actual velocity. Assume a squad of 4-6 mid-to-senior Laravel engineers.

## Milestone 0 — Foundation (Weeks 1-3)

**Goal:** Repo, infrastructure, conventions, skeleton app. No user-facing features.

**Scope:**
- Laravel 13 project with modular structure scaffolded (empty modules + service providers).
- PHP 8.3, MySQL 8, Redis, Elasticsearch, Reverb installed locally via Docker.
- CI pipeline (lint, static analysis, Deptrac, Pest).
- Staging environment deployed from `main`.
- Horizon configured with queue layout from Part 3.
- Base layout with FluxUI + Livewire.
- Shared Kernel v1: `Money`, `Ulid`, `IdempotencyKey`, base exceptions.
- Logging + APM (Sentry or equivalent) wired up.
- Feature flag framework (Pennant) installed.
- `/docs/adr/` seeded with foundational ADRs (modular monolith choice, Stripe Connect choice, etc.).

**Dependencies:** None.

**Risks:** Over-engineering foundations. Keep it minimal and functional.

**Quality gates:**
- CI green end-to-end on a no-op PR.
- `php artisan test` runs Pest suite.
- Staging deploy works on merge.
- Deptrac passes on the empty module structure.

**Acceptance criteria:**
- A new engineer can clone, `./setup.sh`, and be running in <30 minutes.
- A trivial feature PR goes through the full lifecycle in <1 day.

## Milestone 1 — Core Identity (Weeks 4-6)

**Goal:** Users can register, verify, log in, manage basic profile.

**Scope:**
- Registration with email verification.
- Login with rate limiting + breached password check.
- 2FA (TOTP).
- Password reset.
- Session management (device list, revoke).
- Basic profile: avatar, display name, bio, country, languages.
- User roles (buyer default, seller opt-in).
- Global middleware enforcing account restrictions.
- Suspension/restriction data model in place.

**Dependencies:** M0.

**Risks:** Scope creep into full profile features. Defer portfolio, skills to M2.

**Quality gates:**
- Pen-test or security review on auth flows.
- Rate limiting verified against automated signup attempts.
- All auth forms CSRF-protected, XSS-escaped.

**Acceptance criteria:**
- User can complete the full lifecycle: register → verify → log in → enable 2FA → log out → log back in.
- Suspended user cannot perform protected actions.

## Milestone 2 — Seller System (Weeks 7-11)

**Goal:** Sellers can create and manage gigs; the catalog exists.

**Scope:**
- Seller profile expansion: skills, languages, education, certifications, portfolio links.
- Gig creation wizard (5 steps).
- Gig packages, extras, FAQs, requirements questionnaire.
- Gig gallery with File Upload module.
- Gig state machine (draft → pending → active → paused → rejected).
- Category + subcategory catalog (seeded).
- Gig public page (read-only for buyers, will integrate with M3).
- Basic seller workspace dashboard.
- Seller vacation mode.
- Events: `GigCreated`, `GigSubmittedForReview`, `GigApproved`, `GigPaused`.

**Dependencies:** M1, File Upload module.

**Risks:**
- Category metadata modeling — do not build EAV. JSON + per-category validation.
- Image processing pipeline complexity — queue everything, handle failures.

**Quality gates:**
- 100+ gig creation E2E tests passing.
- Image processing survives 10x load test.
- Gig approval flow tested end-to-end (approval by admin event will be stubbed; actual admin UI is the separate app).

**Acceptance criteria:**
- Seller creates a gig, submits for review, admin approves (via seeded data or API call simulating admin app), gig appears live with gallery and packages.

## Milestone 3 — Buyer System (Weeks 12-15)

**Goal:** Buyers can discover and evaluate gigs. Search works.

**Scope:**
- Homepage with category browsing.
- Gig listing pages (category, subcategory).
- Elasticsearch indexing pipeline for gigs (triggered by events from M2).
- Search API + search page with filters (category, price, delivery, seller level, language).
- Autocomplete suggestions.
- Gig detail page (buyer view).
- Saved gigs (wishlist).
- Browsing history.
- Buyer workspace skeleton.

**Dependencies:** M2.

**Risks:**
- Relevance tuning — budget time for iteration. Do not try to ship perfect relevance in M3.
- ES sync bugs — build the reconciliation job early.

**Quality gates:**
- Search returns results in <200ms p95 on seeded dataset.
- ES reconciliation job detects and repairs injected drift.
- Filter counts accurate on sample queries.

**Acceptance criteria:**
- Buyer can search "logo design", filter by $50-100, find relevant gigs, click through, view detail page.

## Milestone 4 — Orders Engine (Weeks 16-21)

**Goal:** Full order lifecycle except payments (stubbed).

**Scope:**
- Checkout flow (cart → checkout → review), payment stubbed.
- Order creation.
- Requirements flow.
- Delivery flow.
- Revision flow.
- Completion flow (manual + auto).
- Cancellation flow (mutual, late, admin-stubbed).
- Full order state machine per Part 4.
- Order events audit table.
- Order page UI (buyer + seller views, same data, different affordances).
- Deadline timer service + sweeper jobs.
- System messages injected into thread (thread entity created — messaging UI comes in M6).

**Dependencies:** M2, M3.

**Risks:**
- State machine complexity — this is the most error-prone milestone. Start with the state machine class and 100% test coverage before wiring to UI.
- Timer edge cases — DST, pauses, concurrent transitions.

**Quality gates:**
- State machine has 100% branch coverage.
- All transitions from Part 4 have explicit test cases.
- All invalid transitions throw.
- Sweeper job runs idempotently under simulated race conditions.

**Acceptance criteria:**
- Complete happy path: buyer orders → submits requirements → seller delivers → buyer accepts. Completed in test.
- Revision path: seller delivers → buyer requests revision → seller redelivers → auto-completes.
- All edge cases from 4.5 covered.

## Milestone 5 — Payments (Weeks 22-27)

**Goal:** Real money moves. Orders are monetized end-to-end.

**Scope:**
- Stripe Connect onboarding for sellers (Express accounts).
- Payment authorization at checkout (Stripe Payment Elements).
- Webhook handling with idempotency and outbox.
- Ledger schema + LedgerService (the only writer).
- Wallet accounts derived from ledger.
- Escrow → clearing → available flow.
- Withdrawal requests + payouts via Stripe.
- Refunds (to original method + to wallet).
- Reconciliation jobs (payments vs ledger, Stripe vs ledger).
- Currency handling (USD single-currency in v1; multi-currency deferred).
- Stripe webhooks replay tooling.

**Dependencies:** M4.

**Risks:**
- This is the highest-risk milestone. Budget 20% of time for hardening.
- Chargeback handling — do not defer; build the claw-back flow in this milestone.
- Compliance — KYC requirements for payouts (Stripe handles most, but gaps exist).

**Quality gates:**
- Every transaction reconciles with Stripe (automated daily check).
- Simulated chargeback correctly claws back seller balance.
- Double-click / double-webhook tests pass (no double charges, no double refunds).
- Security review on all Stripe integration.

**Acceptance criteria:**
- Buyer pays real money (test mode), seller onboards, order completes, seller withdraws to their test Stripe account, ledger balances to zero.

## Milestone 6 — Messaging + Notifications (Weeks 28-32)

**Goal:** Real communication between users. Multi-channel notifications.

**Scope:**
- Messaging module: threads, messages, attachments.
- Content scanning pipeline (regex v1; ML deferred).
- Real-time messaging via Reverb.
- Online presence + typing indicators.
- Custom offers as message type.
- Buyer Requests module (brief inbox).
- Notifications module: in-app, email, push (web push v1; mobile native later).
- Notification preferences + digest batching.
- System messages on order events.

**Dependencies:** M4, M5 (custom offers create orders).

**Risks:**
- WebSocket scaling — load test Reverb to 10k concurrent connections.
- Message abuse patterns — content scanning will miss things; build reporting flow.

**Quality gates:**
- Messages deliver in <500ms p95 under simulated 1000 concurrent users.
- Contact info scanning blocks known patterns (unit-tested corpus).
- Notification preferences respected (no emails sent when user opted out).

**Acceptance criteria:**
- Two users message in real time with attachments.
- A sent message containing an email gets flagged.
- A custom offer accepted creates an order via the normal flow.

## Milestone 7 — Reviews + Trust (Weeks 33-36)

**Goal:** Reviews, seller levels, trust signals.

**Scope:**
- Reviews module with multi-axis ratings.
- Seller responses to reviews.
- Aggregation (denormalized + nightly recompute).
- Seller level calculator + nightly job.
- Trust & Safety module with fraud scoring on reviews.
- Verified badges (ID, phone).
- Re-ranking in search using seller levels + ratings.

**Dependencies:** M4 (orders complete → review allowed), M3 (search re-ranking).

**Risks:** Aggregation drift — build the nightly reconcile and alert on drift > 0.1.

**Quality gates:**
- Review → aggregate update happens in the same transaction.
- Nightly recompute matches live aggregates on clean data.
- Seller level transitions tested with rolling-window fixtures.

**Acceptance criteria:**
- Completed order → review submitted → seller rating updates immediately.
- Seller crosses threshold → auto-promoted to Level 1 → gig search ranking reflects it.

## Milestone 8 — Disputes + Refunds (Weeks 37-40)

**Goal:** Formal conflict resolution. Refund abuse mitigations.

**Scope:**
- Dispute Center module.
- Proposal/counter-proposal flow with 48h timers.
- Dispute → resolution → enacting state changes on order.
- Escalation event published for admin app.
- Refund abuse detection (buyer cancellation rate tracked, trust score drops).
- Enhanced cancellation reasons taxonomy.
- Admin-stubbed endpoints for dispute resolution (actual UI is in admin app).

**Dependencies:** M4, M5, M7.

**Risks:** 48h timer bugs — same paused-timer discipline as Part 4.

**Quality gates:**
- All dispute paths (accept, reject, escalate, timeout) tested end-to-end.
- Refund triggered by dispute executes correctly against Stripe in test mode.

**Acceptance criteria:**
- Buyer opens dispute with full-refund proposal → seller accepts → order cancels → refund issued → ledger balances.
- Timeout path: seller doesn't respond in 48h → dispute escalates.

## Milestone 9 — Optimization + Scaling (Weeks 41-46)

**Goal:** Production-ready performance and resilience.

**Scope:**
- Load test (10x expected day-1 traffic).
- Database query optimization (slow-query log review, missing indexes).
- Caching layer for hot reads (gig page, category page).
- CDN for static assets + signed URLs for private uploads.
- ES query tuning + result caching.
- Horizon tuning + queue priority review under load.
- Observability dashboards (order lifecycle funnel, payment success rate, search latency).
- Runbooks for top 10 incident scenarios.
- Disaster recovery drill.
- Security audit (external).

**Dependencies:** M1-M8.

**Risks:** Identifying real bottlenecks vs imagined ones — use actual load testing data.

**Quality gates:**
- 10x load test passes SLOs (p95 latency budgets by page).
- DR drill restores service within RTO.
- External security audit results addressed (high + critical).

**Acceptance criteria:**
- Platform survives a simulated traffic spike equivalent to a marketing push.
- On-call engineer can resolve a top-10 incident using the runbook alone.

---

# PART 7 — FUTURE RISKS & FAILURE PREDICTION

These are the failure modes that will hit us. Listed with their warning signs and prevention strategies.

## 7.1 Architectural risks

**Module boundary erosion.** A junior engineer is told "just import that class, it's faster". Over months, cross-module dependencies accrete. Eventually the "modules" are walls with holes.

- *Prevention*: Deptrac in CI, Shared Kernel has two-approver rule, every cross-module dependency requires an ADR.

**Shared Kernel bloat.** Everyone adds "just a utility" to SharedKernel. It becomes `CommonHelpers v3`.

- *Prevention*: The two-approver rule + explicit criterion (used by ≥3 modules AND stable).

**Service class sprawl.** A new team member creates `OrderService` because `OrderStateMachine` "feels wrong". Now there are two.

- *Prevention*: One canonical service per concept, enforced by code review + architecture docs listing the canonical services.

**Premature microservice extraction.** Someone says "we should extract Search to its own service" at month 12.

- *Prevention*: ADR required. Microservice extraction requires: (a) proven scale need, (b) a team to own it, (c) ops maturity. Default answer: no.

## 7.2 Scaling risks

**Hot gigs table.** Trending gigs cause MySQL hot rows. Gig page reads pile up.

- *Prevention*: Cache gig page fragments in Redis with tag-based invalidation on `GigUpdated`. Read-through cache.

**Orders table growth.** Millions of rows. Queries on `seller_id + status` slow down.

- *Prevention*: Composite indexes on all common filter combinations. Archive orders older than N years to a cold table. Partition by created_at (monthly) if growth demands.

**Message table growth.** Messages grow fastest. Thread lookups degrade.

- *Prevention*: Partition `messages` by month. Cold threads archived to S3 via Parquet for analytics.

**WebSocket connection limits.** Single Reverb node caps at ~10-20k connections.

- *Prevention*: Horizontal Reverb scaling with Redis pub/sub. Plan the cutover early; the operational complexity is non-trivial.

**N+1 in Livewire components.** Component renders inside a loop, each hydration causes its own queries.

- *Prevention*: `Model::preventLazyLoading()` enabled in non-prod. Laravel Debugbar in local. Static rule banning `@livewire` inside `@foreach` without `wire:key` and data-passed-in.

## 7.3 Payment risks

**Double charging.** User double-clicks checkout. Without idempotency, two charges.

- *Prevention*: Idempotency-Key header mandatory on checkout; frontend generates one per checkout session.

**Webhook delivery failures.** Stripe retries for 3 days; we miss an event.

- *Prevention*: Idempotent webhook handlers + the outbox pattern + daily reconciliation job that compares Stripe events list with our processed events.

**Race on refund.** Two refund requests for the same order.

- *Prevention*: Refunds are keyed by order + reason. Unique constraint. Second request returns the first's result.

**Ledger drift.** A bug writes an unbalanced pair of entries.

- *Prevention*: The `LedgerService` enforces balance at the API level. Daily check: `SUM(amount) GROUP BY order_id` must equal zero for completed orders. Alert on any drift.

**Stripe account deletion / change by seller.** Seller changes their bank; pending payout fails.

- *Prevention*: Payout state machine with retry logic. On repeated failure, notify seller and hold funds.

**Chargeback cascade.** A fraudster buyer chargebacks 10 orders. Seller balance goes negative.

- *Prevention*: Freeze buyer on first confirmed chargeback. Pattern detection on seller side — if >N chargebacks on one seller, review seller. Wallet can go negative; withdrawal blocked until cleared.

## 7.4 Fraud risks

**Off-platform payment solicitation.** Seller tells buyer "pay me on PayPal, I'll give 30% off".

- *Prevention*: Content scanning, user reports button, aggressive enforcement on confirmed cases (instant suspension, forfeited balance). Education in seller onboarding.

**Fake portfolios.** Seller uses stolen work.

- *Prevention*: Reverse image search at gig approval (integrate with an image similarity service). DMCA takedown process. Buyer reports.

**Sockpuppet reviews.** Seller orders from their own gig via a fake buyer account.

- *Prevention*: Buyer account quality score weighted into review visibility. Same-device, same-IP, same-payment-method detection. Review fraud scoring async job.

**Account resale.** Someone sells an aged Level 2 account.

- *Prevention*: Device fingerprinting, KYC re-verification required for first withdrawal over a threshold. Anomaly detection on sudden login country changes.

**Refund abuse.** Buyer orders, gets delivery, cancels claiming non-delivery.

- *Prevention*: Delivery evidence requirement (files or screen recordings). Buyer trust score drops on each dispute loss. Cap on refund rate — above threshold, disputes require evidence review.

## 7.5 Race conditions and concurrency

**Concurrent order transitions.** Buyer accepts while auto-complete fires.

- *Prevention*: Pessimistic lock on order row in state machine.

**Concurrent withdrawal requests.** User opens two tabs, clicks withdraw in both.

- *Prevention*: Redis lock per wallet on withdrawal initiation + DB unique constraint on `(wallet_id, status)` where status is "processing".

**Coupon over-redemption.** Two users use the last available slot of a 100-use coupon simultaneously.

- *Prevention*: Pessimistic lock on coupon row during validation + redemption.

**Gig edit during order.** Seller edits gig while buyer is at checkout.

- *Prevention*: Orders snapshot gig state at creation. Checkout re-validates price at capture — if it changed, abort and show the new price to the buyer.

**Counter update races.** Concurrent orders increment `seller.total_orders` — lost updates.

- *Prevention*: Counters updated via atomic `UPDATE ... SET total_orders = total_orders + 1`. Better: don't store counters — derive from `COUNT(*)` with caching, or use Redis `INCR`.

## 7.6 Queue failures

**Worker crash mid-job.** Job partially executed, retries cause duplicate effects.

- *Prevention*: Idempotency in every job. DB transactions scope the side effects of non-idempotent sections.

**Queue backlog.** Search queue backs up during a reindex, order processing starves.

- *Prevention*: Separate queues per concern (see Part 3.6). Horizon alerts on queue lag > 30s.

**Poison message.** A malformed payload crashes the job repeatedly.

- *Prevention*: Bounded retries + dead-letter table. Alerts on DLQ arrivals.

**Scheduled job overlap.** Cron fires while previous run still executing.

- *Prevention*: `withoutOverlapping()` on schedule + sensible timeouts.

## 7.7 Notification failures

**Email not delivered.** Bounces, spam filter, ESP outage.

- *Prevention*: ESP with high deliverability (Postmark/SES). Monitor bounce rate. Fall back to secondary ESP on primary outage. Critical notifications (payment confirmation) also appear in-app.

**Notification loop.** A poorly-written listener fires an event that triggers itself.

- *Prevention*: Events must not be fired from within listeners of the same event. Linting rule. If cross-event emission is needed, use an explicit intermediate state.

**Notification spam.** Bug causes 1000 emails per user.

- *Prevention*: Rate limit per user per notification type (e.g., max 1 of the same type per 5 minutes). Kill switch to disable a specific notification type fast.

## 7.8 Elasticsearch sync issues

**Missed event.** A `GigUpdated` event is fired but the listener fails silently — ES out of date.

- *Prevention*: Inbox pattern for the indexer (records processed event IDs). Nightly reconcile compares doc counts + spot-checks random gigs.

**Reindex during production.** Full reindex locks the cluster.

- *Prevention*: Blue/green index strategy with alias flip. Bulk operations rate-limited.

**Mapping change breaks search.** Schema evolution mishandled.

- *Prevention*: Mapping changes require ADR + new index version + tested migration job + canary traffic before full switch.

## 7.9 Messaging consistency

**System messages lost.** Order delivery event fires, but the system message to the thread fails to write.

- *Prevention*: System message write is part of the order transition's listener, enforced by the inbox pattern so retries converge. If the thread write fails persistently, it enters DLQ with the event ID for manual replay.

**Out-of-order messages.** Two messages written in the same millisecond, UI shows them in wrong order.

- *Prevention*: Monotonic sequence per thread, not timestamp. Returned by DB trigger or by `SELECT MAX(sequence) FOR UPDATE` within the insert transaction.

## 7.10 Seller abuse patterns

**Bid-and-abandon.** Seller accepts orders they can't deliver, then cancels en masse.

- *Prevention*: Cancellation rate metric affects level + search ranking. Hard cap on concurrent active orders by level.

**Review gaming via discount.** Seller offers a partial refund in exchange for a 5-star review.

- *Prevention*: Content scan in messaging for "review + refund" patterns. Reports followed up seriously.

**Gig spam.** One seller creates 50 near-identical gigs to dominate search.

- *Prevention*: Duplicate detection at gig submission. Per-seller active gig cap per category.

## 7.11 Buyer abuse patterns

**Requirements harassment.** Buyer submits requirements, then repeatedly "updates" them to delay.

- *Prevention*: Requirements are submit-once. Changes require an explicit seller-approved revision.

**Drive-by cancellation.** Buyer orders, then cancels within 1 hour repeatedly.

- *Prevention*: Cancellation rate tracking. High rate → trust score drop → restrictions on new orders without payment hold.

**Review blackmail.** "Give me a refund or I leave a 1-star."

- *Prevention*: Content scanning. Reports escalate. Review integrity review before posting (holds for X hours if flagged).

## 7.12 Refund abuse patterns

**Friendly fraud.** Buyer receives delivery, chargebacks card.

- *Prevention*: Strong evidence package submitted to Stripe (delivery files, messages, timestamps). Auto-generate evidence on chargeback.

**Multi-account refunds.** User refunds on account A, creates account B, repeats.

- *Prevention*: Device fingerprinting + payment method fingerprinting. Refund history follows the device/card, not just the account.

---

## Appendix A — High-risk invariants the platform MUST preserve

1. Every order has a complete audit trail in `order_events`.
2. Every cent movement has two matching ledger entries in the same transaction.
3. No operation that moves money runs without an idempotency key.
4. No state transition runs without going through `OrderStateMachine::transition`.
5. No cross-module import exists outside `Contracts/` and `SharedKernel/`.
6. No business logic lives in Blade templates, controllers, or model classes.
7. Every listener that matters is idempotent via the inbox pattern.
8. Every external call (Stripe, ESP, ES) happens outside the DB transaction that triggered it.
9. No float is ever used to represent money.
10. Every migration has a tested `down()` or an explicit ADR waiving it.

## Appendix B — Recommended packages

- `laravel/horizon`
- `laravel/reverb`
- `laravel/pennant` (feature flags)
- `laravel/pulse` (app monitoring)
- `spatie/laravel-model-states` (state machines)
- `spatie/laravel-activitylog`
- `qossmic/deptrac` (module boundaries)
- `larastan/larastan` (PHPStan for Laravel)
- `pestphp/pest`
- `stripe/stripe-php`
- `elastic/elasticsearch` (the PHP client)
- `brick/money` (if we want a battle-tested Money library rather than rolling our own)

## Appendix C — What NOT to build in v1

Keep ruthless discipline on scope. These are seductive but non-essential:

- Multi-currency pricing (single USD in v1).
- Mobile apps (PWA is enough at launch).
- Video calls between buyer/seller.
- AI-powered anything on the critical path.
- Blog/content marketing platform in-app.
- Affiliate program.
- Promoted gigs (ads).
- Multi-language gigs.
- Advanced analytics for sellers beyond basic counts.
- Gift cards.

These are real features, but they're v2/v3 material.

---

**End of document.**

*Maintained in `/docs/architecture/strategy.md`. Updates require an ADR.*

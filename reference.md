<!--
Mirrored from: https://github.com/outegram/market/blob/main/milestone-issues.md
This is an offline reference pinned at the time the outegram-market-milestones
skill was created. The GitHub copy is the live source of truth; re-sync this
file if the upstream doc changes.
-->

# Milestone Issues — Complete Reference

**Project:** Marketplace Platform
**Total milestones:** 9 (M0 → M8)
**Total issues:** 57
**Document status:** Reference — matches GitHub milestones and issues
**Location:** `docs/process/MILESTONE_ISSUES.md`

Each issue maps to exactly one GitHub Issue, one branch, one PR.
Issues within a milestone may be worked in parallel unless a dependency is noted.
No milestone opens until all issues in the previous milestone are merged and closed.

---

## Quick reference

| Milestone | Name | Issues | Dependency |
|---|---|---|---|
| M0 | Infrastructure & Architecture | 8 | None |
| M1 | User + Taxonomy + Service | 10 | M0 complete |
| M2 | Orders + Messaging + Reviews | 9 | M1 complete |
| M3 | Payments + Billing | 7 | M2 complete + zero open architecture-exceptions |
| M4 | Notifications + Analytics | 5 | M3 complete |
| M5 | Search | 5 | M1 complete (can run parallel to M2–M4 after M1) |
| M6 | Admin Panel | 5 | M3 complete |
| M7 | Stabilization | 5 | M6 complete |
| M8 | Production Launch | 3 | M7 complete |

---

---

# Milestone 0 — Infrastructure & Architecture

**Gate condition:** All 8 issues closed AND Technical Approver commits
sign-off to `docs/architecture/milestone-0-execution-contract.md`.

**Labels:** `devops`, `backlog`

---

### M0-01 — Repository initialisation and local environment

**Branch:** `chore/m0-01-repo-init`
**Labels:** `devops`, `backlog`

**Objective:**
Initialise the Laravel 13 project with a clean local development environment
that every developer can reproduce in under 15 minutes.

**Tasks:**
- [ ] Run `laravel new marketplace-platform` on PHP 8.3 — clean install, no errors
- [ ] Configure Laravel Sail: add MySQL 8, Redis 7, Elasticsearch 8 to `docker-compose.yml`
- [ ] Verify `sail up` starts all three services without errors
- [ ] Verify `php artisan migrate` runs on a fresh database without errors
- [ ] Commit `.env.example` with all required keys (see Section 16 of engineering contract)
- [ ] Commit `.editorconfig` with consistent indentation and line ending rules
- [ ] Commit `Makefile` with: `make up`, `make down`, `make test`, `make lint`, `make analyse`, `make fresh`
- [ ] Update `README.md` with complete local setup instructions

**Acceptance criteria:**
- A developer who clones the repo and runs `make up && make fresh && make test` gets green output
- `.env.example` contains every key that the application needs — no undocumented environment variables

---

### M0-02 — Code quality tooling

**Branch:** `chore/m0-02-code-quality`
**Labels:** `devops`, `backlog`
**Depends on:** M0-01

**Objective:**
Enforce consistent code style and static analysis across the entire project
before any application code is written.

**Tasks:**
- [ ] Install Laravel Pint: `composer require laravel/pint --dev`
- [ ] Commit `pint.json` with PSR-12 ruleset
- [ ] Verify `./vendor/bin/pint --test` exits 0 on a fresh install
- [ ] Install PHPStan + Larastan: `composer require nunomaduro/larastan phpstan/phpstan --dev`
- [ ] Commit `phpstan.neon` configured at level 6 with Larastan extension
- [ ] Verify `./vendor/bin/phpstan analyse` exits 0 on a fresh install
- [ ] Commit `phpstan-architecture.neon` — separate config for custom architecture rules
- [ ] Install and configure Husky pre-commit hook (runs Pint on commit)
- [ ] Install and configure pre-push hook (runs PHPStan + Pest before push)
- [ ] Add `make lint` and `make analyse` shortcuts to `Makefile`

**Acceptance criteria:**
- Pre-commit hook blocks a commit that has Pint violations
- PHPStan passes at level 6 with zero errors on a fresh install
- A deliberate formatting violation fails `make lint`

---

### M0-03 — CI/CD pipeline

**Branch:** `chore/m0-03-ci-pipeline`
**Labels:** `devops`, `backlog`
**Depends on:** M0-02

**Objective:**
Every pull request passes automated checks before it can merge.
CI passes on the default branch with zero application code.

**Tasks:**
- [ ] Commit `.github/workflows/ci.yml` with six jobs in order:
  - `install` — Composer install with dependency caching
  - `lint` — Laravel Pint formatting check
  - `analyse` — PHPStan level 6
  - `test` — Pest suite with MySQL 8 and Redis service containers
  - `docs-freshness` — fails if architecture docs untouched > 90 days
  - `architecture-rules` — custom PHPStan rules via `phpstan-architecture.neon`
- [ ] Verify CI passes on a fresh Laravel install with zero application code
- [ ] Enable branch protection on `main` per `.github/BRANCH_PROTECTION.md`
- [ ] Verify direct push to `main` is blocked
- [ ] Verify PR cannot merge until all six CI jobs pass
- [ ] Commit `.github/CODEOWNERS` with all path assignments
- [ ] Set up GitHub Actions secrets: `APP_KEY` for test environment

**Acceptance criteria:**
- A PR with a Pint violation fails CI at the `lint` job
- A PR with a PHPStan error fails CI at the `analyse` job
- A PR with a failing test fails CI at the `test` job
- Direct push to `main` is rejected by GitHub

---

### M0-04 — Architecture custom PHPStan rules

**Branch:** `chore/m0-04-phpstan-rules`
**Labels:** `devops`, `backend`, `backlog`
**Depends on:** M0-02

**Objective:**
Automate enforcement of the two most critical architecture rules so CI fails
on violation without requiring human review.

**Tasks:**
- [ ] Implement PHPStan rule: `AtomicActionMustNotCallOtherActions`
  - Scans all classes in `Actions/Atomic/`
  - Fails if any method instantiates or resolves another Action class
  - Fails if any method calls `dispatch()` on a Job
  - Fails if any method calls `event()` or `Event::dispatch()`
- [ ] Implement PHPStan rule: `OnlyOrchestratorsMayStartTransactions`
  - Scans all classes in `Actions/Atomic/`
  - Fails if any method calls `DB::transaction()`, `DB::beginTransaction()`, or `DB::commit()`
- [ ] Commit both rules to `app/PHPStan/Rules/`
- [ ] Register both rules in `phpstan-architecture.neon`
- [ ] Write a test that deliberately violates each rule and confirm CI fails
- [ ] Write a test that follows the rule correctly and confirm CI passes
- [ ] Add both rules to the CI `architecture-rules` job in `ci.yml`

**Acceptance criteria:**
- An Atomic Action calling another Action fails the `architecture-rules` CI job
- An Atomic Action calling `DB::transaction()` fails the `architecture-rules` CI job
- A correct Atomic Action passes both rules without errors

---

### M0-05 — Architecture documentation and governance files

**Branch:** `chore/m0-05-architecture-docs`
**Labels:** `documentation`, `backlog`
**Depends on:** M0-01

**Objective:**
Commit all architecture governance documents before any application code is written.
These are the reference every developer uses during implementation.

**Tasks:**
- [ ] Commit `docs/architecture/PROJECT_RULES_AND_EXECUTION.md` — v1.1, frozen
- [ ] Commit `docs/architecture/where-does-this-go.md` — decision reference card
- [ ] Commit `docs/architecture/review-checklist.md` — reviewer protocol
- [ ] Commit `docs/architecture/rules.md` — 11 rules summary
- [ ] Commit `docs/architecture/elasticsearch-ops.md` — operational runbook
- [ ] Commit `docs/architecture/onboarding-guide.md` — beginner-friendly guide
- [ ] Commit `docs/architecture/milestone-0-execution-contract.md` — with all three role slots filled
- [ ] Commit `.github/pull_request_template.md`
- [ ] Commit `CONTRIBUTING.md`
- [ ] Verify all three role slots in the execution contract are filled with real names/handles
- [ ] Verify the documentation freshness CI check passes for all committed docs

**Acceptance criteria:**
- All seven architecture docs are committed and accessible via `docs/architecture/`
- The execution contract has Technical Approver, Reviewer, and Architecture Steward named
- `make analyse` and the CI docs-freshness check both pass

---

### M0-06 — GitHub project setup (labels, milestones, issues)

**Branch:** `chore/m0-06-github-setup`
**Labels:** `devops`, `backlog`
**Depends on:** M0-05

**Objective:**
Create all GitHub labels, milestones, and issues so the entire project
backlog is visible and organised before implementation begins.

**Tasks:**
- [ ] Run label creation script from `.github/LABELS.md` — all 19 labels created
- [ ] Verify `architecture-exception` label exists (Architecture Steward confirms)
- [ ] Run milestone creation script from `.github/MILESTONES.md` — all 9 milestones created
- [ ] Create all 57 issues from this document in GitHub
- [ ] Apply correct labels and milestones to each issue
- [ ] Create GitHub Project board with columns: Backlog → To Do → In Progress → In Review → Done
- [ ] Add all issues to the project board
- [ ] Assign M0 issues to team members

**Acceptance criteria:**
- All 19 labels exist in the repository
- All 9 milestones exist with due dates
- All 57 issues exist, labelled, and assigned to a milestone
- Project board is visible and all issues appear in Backlog column

---

### M0-07 — Shared Kernel foundation

**Branch:** `feature/m0-07-shared-kernel`
**Labels:** `backend`, `backlog`
**Depends on:** M0-01

**Objective:**
Create the Shared Kernel with value objects and cross-domain contracts.
This is the vocabulary that every domain will depend on.

**Tasks:**
- [ ] Create `app/Shared/ValueObjects/Money.php`
  - Constructor: `(int $amount, CurrencyCode $currency)`
  - Methods: `add()`, `subtract()`, `multiply()`, `percentage()`, `formatted()`
  - Immutable — all arithmetic returns a new instance
  - Throws `InvalidArgumentException` on negative amount
- [ ] Create `app/Shared/ValueObjects/EmailAddress.php`
  - Validates format on construction — throws on invalid
  - `value()` returns the string
- [ ] Create `app/Shared/ValueObjects/PhoneNumber.php`
  - Validates E.164 format on construction
  - `value()` returns the normalised string
- [ ] Create `app/Shared/Enums/CurrencyCode.php` — USD, EUR, GBP, SEK
- [ ] Create `app/Shared/Contracts/Actions/ActionInterface.php` — `execute(): mixed`
- [ ] Create `app/Shared/Contracts/Infrastructure/PaymentGatewayInterface.php`
- [ ] Create `app/Shared/Contracts/Infrastructure/SearchIndexInterface.php`
- [ ] Create `app/Shared/Contracts/Infrastructure/FileStorageInterface.php`
- [ ] Create `app/Shared/Contracts/Infrastructure/NotificationChannelInterface.php`
- [ ] Create `app/Providers/InfrastructureServiceProvider.php` — empty, ready for bindings
- [ ] Create `app/Providers/EventServiceProvider.php` — empty event map, ready for listeners
- [ ] Unit tests for all three value objects — valid construction, invalid construction, arithmetic

**Acceptance criteria:**
- `Money::add()` returns a new `Money` instance — original is unchanged
- `Money` constructed with negative amount throws `InvalidArgumentException`
- `EmailAddress` constructed with invalid email throws
- `PhoneNumber` constructed with non-E.164 string throws
- All value object tests pass

---

### M0-08 — Reference implementations

**Branch:** `feature/m0-08-reference-implementations`
**Labels:** `backend`, `backlog`
**Depends on:** M0-07, M0-04, M0-05

**Objective:**
Write the three reference implementations that define the architectural standard
for the entire codebase. Every future Action is measured against these files.
Completed sequentially — next does not start until previous is merged.

**Tasks — Reference 1: RegisterUserAction**
- [ ] Create `app/Domain/User/Enums/UserRole.php` (buyer, seller, admin)
- [ ] Create `app/Domain/User/Enums/UserStatus.php` (active, suspended, banned, pending)
- [ ] Create User Eloquent model (minimal — email, password, role, status)
- [ ] Create `RegisterUserDTO` with `fromRequest()` factory
- [ ] Create `app/Domain/User/Actions/Atomic/CreateUserRecordAction.php`
- [ ] Create `app/Domain/User/Actions/Atomic/HashPasswordAction.php`
- [ ] Create `app/Domain/User/Actions/Orchestrators/RegisterUserAction.php`
- [ ] Create `app/Domain/User/Events/UserRegisteredNotificationEvent.php`
- [ ] Integration test: `RegisterUserAction` creates user, fires event
- [ ] Verbal explanation test passed — documented in PR
- [ ] File header: `// REFERENCE IMPLEMENTATION`

**Tasks — Reference 2: CreateServiceAction**
- [ ] Create `app/Domain/Service/Enums/ServiceStatus.php`
- [ ] Create `MarketplaceService` Eloquent model (minimal)
- [ ] Create `app/Domain/Service/Actions/Atomic/CreateServiceRecordAction.php`
- [ ] Create `app/Domain/Service/Actions/Atomic/AttachServiceSkillsAction.php`
- [ ] Create `app/Domain/Service/Actions/Orchestrators/CreateServiceAction.php`
- [ ] Create `app/Domain/Service/Policies/ServicePolicy.php`
- [ ] Create `app/Domain/Service/Events/ServiceCreatedNotificationEvent.php`
- [ ] Integration test: `CreateServiceAction` creates service, respects policy, fires event
- [ ] Verbal explanation test passed — documented in PR
- [ ] File header: `// REFERENCE IMPLEMENTATION`

**Tasks — Reference 3: PlaceOrderAction**
- [ ] Create `app/Domain/Order/Enums/OrderStatus.php`
- [ ] Create `Order` Eloquent model (minimal)
- [ ] Create `PlaceOrderDTO`
- [ ] Create `app/Domain/Order/Actions/Atomic/CreateOrderRecordAction.php`
- [ ] Create `app/Domain/Order/Actions/Atomic/SnapshotPackagePricingAction.php`
- [ ] Create `app/Domain/Order/Events/OrderPlacedNotificationEvent.php`
- [ ] Create `app/Domain/Order/Actions/Orchestrators/PlaceOrderAction.php`
  - DB transaction opened here
  - Calls `CreateOrderRecordAction` then `SnapshotPackagePricingAction`
  - Fires `OrderPlacedNotificationEvent` with `$afterCommit = true` on listener
- [ ] Integration test — success path: order created, pricing snapshotted, event fired
- [ ] Integration test — failure path: transaction rolls back, no order record written
- [ ] Verbal explanation test passed — documented in PR
- [ ] File header: `// REFERENCE IMPLEMENTATION`

**Acceptance criteria:**
- All three Actions are merged sequentially (not simultaneously)
- All tests pass
- Verbal explanation test passed for all three — documented in each PR
- Technical Approver has given final approval on all three
- Execution contract sign-off committed to `milestone-0-execution-contract.md`

---

---

# Milestone 1 — User + Taxonomy + Service

**Gate condition:** All M0 issues closed. Execution contract signed off.

**Labels:** `backend`, `backlog`

---

### M1-01 — All database migrations

**Branch:** `db/m1-01-all-migrations`
**Labels:** `database`, `backlog`

**Objective:**
Create all 43 database migrations in the correct dependency order.
No application logic — migrations only.

**Tasks:**

*Taxonomy tables (no FKs — create first):*
- [ ] `create_countries_table` (code PK, name, region, flag_emoji, is_active)
- [ ] `create_languages_table` (code PK, name, is_active)
- [ ] `create_categories_table` (id, parent_id self-join, name, slug, icon_url, sort_order, is_active)
- [ ] `create_skills_table` (id, slug unique, name, category_id FK, usage_count, is_active)
- [ ] `create_tags_table` (id, slug unique, name, usage_count, is_admin_curated, is_active)

*Identity tables:*
- [ ] `create_users_table` (all fields per master structure — role enum, status enum, seller_level enum, balance int, stripe ids, soft deletes)
- [ ] `create_user_social_accounts_table`
- [ ] `create_seller_onboarding_applications_table` (unique user_id, step enum, status enum, JSON columns for each step)
- [ ] `create_seller_profiles_table` (unique user_id FK, all KPI fields)
- [ ] `create_seller_skills_table` (pivot: user_id, skill_id, level enum)
- [ ] `create_seller_languages_table` (pivot: user_id, language_code, proficiency enum)
- [ ] `create_user_devices_table` (push tokens)
- [ ] `create_notification_preferences_table`
- [ ] `create_saved_services_table` (pivot)
- [ ] `create_feature_flags_table`

*Service listing tables:*
- [ ] `create_marketplace_services_table`
- [ ] `create_service_packages_table`
- [ ] `create_service_extras_table`
- [ ] `create_service_media_table`
- [ ] `create_service_faqs_table`
- [ ] `create_service_requirements_table`
- [ ] `create_service_skills_table` (pivot)
- [ ] `create_service_tags_table` (pivot)
- [ ] `create_service_views_table`
- [ ] `create_buyer_requests_table`
- [ ] `create_custom_offers_table`

*Transaction tables:*
- [ ] `create_orders_table`
- [ ] `create_order_deliveries_table`
- [ ] `create_order_revisions_table`
- [ ] `create_order_requirements_responses_table`
- [ ] `create_order_resolutions_table`
- [ ] `create_conversations_table`
- [ ] `create_messages_table`
- [ ] `create_reviews_table`

*Financial tables (immutable — no SoftDeletes, no updated_at on ledger):*
- [ ] `create_payments_table`
- [ ] `create_escrow_entries_table`
- [ ] `create_withdrawals_table`
- [ ] `create_ledger_entries_table` (no updated_at, no soft deletes — immutable audit trail)
- [ ] `create_platform_credits_table`

*Operational tables:*
- [ ] `create_notifications_table` (Laravel morph, UUID PK)
- [ ] `create_fraud_signals_table`
- [ ] `create_admin_actions_table` (immutable — no updated_at, no soft deletes)
- [ ] `create_search_queries_table`

- [ ] Verify `php artisan migrate:fresh` runs all 43 migrations without errors
- [ ] Verify all FK constraints resolve correctly

**Acceptance criteria:**
- `php artisan migrate:fresh` completes without errors
- `php artisan migrate:fresh --seed` completes without errors (seeds from M1-02)
- `ledger_entries` and `admin_actions` tables have no `updated_at` column (verified)

---

### M1-02 — Taxonomy seeders

**Branch:** `db/m1-02-taxonomy-seeders`
**Labels:** `database`, `backlog`
**Depends on:** M1-01

**Objective:**
Seed the controlled vocabulary tables so the platform has valid taxonomy data
that all other domains can reference.

**Tasks:**
- [ ] `CountrySeeder` — all 249 ISO countries with correct region groupings
- [ ] `LanguageSeeder` — top 40 spoken languages with BCP-47 codes
- [ ] `CategorySeeder` — 10 top-level categories:
  - Graphics & Design
  - Programming & Tech
  - Digital Marketing
  - Writing & Translation
  - Video & Animation
  - Music & Audio
  - Business
  - Data
  - Photography
  - AI Services
  Each with 6–10 subcategories
- [ ] `SkillSeeder` — minimum 200 skills, each mapped to a category via `category_id`
- [ ] Register all seeders in `DatabaseSeeder.php` in correct run order
- [ ] Verify `php artisan db:seed` runs without errors
- [ ] Verify `countries` has 249 rows
- [ ] Verify `categories` has correct parent_id relationships (subcategories point to parent)

**Acceptance criteria:**
- `php artisan migrate:fresh --seed` produces a populated, consistent database
- Skills can be looked up by category (`Skill::where('category_id', $categoryId)->get()` works)
- All 10 top-level categories have at least 6 subcategories

---

### M1-03 — Authentication

**Branch:** `feature/m1-03-authentication`
**Labels:** `backend`, `security`, `backlog`
**Depends on:** M1-01

**Objective:**
Token-based authentication with email verification.
This issue includes all auth endpoints. OAuth is a separate issue.

**Tasks:**
- [ ] Install and configure Laravel Sanctum
- [ ] Expand `User` model with all fields, casts, and relationship stubs
- [ ] `StoreUserRequest` FormRequest with full validation rules
- [ ] `RegisterUserAction` expanded from M0-08 reference — adds Sanctum token return
- [ ] `LoginRequest` FormRequest
- [ ] `AuthController::register()` — calls `RegisterUserAction`, returns user + token
- [ ] `AuthController::login()` — validates credentials, returns Sanctum token
- [ ] `AuthController::logout()` — revokes current token
- [ ] Email verification: `GET /api/v1/auth/verify-email/{id}/{hash}`
- [ ] Resend verification: `POST /api/v1/auth/resend-verification` (throttled 1/min)
- [ ] `EnsureEmailVerified` middleware — blocks transactional routes for unverified users
- [ ] `UserRegisteredNotificationEvent` listener: `SendEmailVerificationListener` (queued)
- [ ] Pest: register success, duplicate email 422, login success, wrong password 401, logout, email verification flow

**Acceptance criteria:**
- Registration returns `{user, token}` with HTTP 201
- Login with wrong password returns HTTP 401
- Unverified user hitting an order endpoint receives HTTP 403
- Token revoked on logout — subsequent requests with that token return HTTP 401

---

### M1-04 — OAuth social login

**Branch:** `feature/m1-04-oauth`
**Labels:** `backend`, `security`, `backlog`
**Depends on:** M1-03

**Objective:**
Allow users to register and log in via Google, GitHub, and LinkedIn without a password.

**Tasks:**
- [ ] Install Laravel Socialite: `composer require laravel/socialite`
- [ ] `UserSocialAccount` model with relationships
- [ ] `SocialAuthController::redirect()` — redirects to provider
- [ ] `SocialAuthController::callback()` — handles provider response
- [ ] OAuth logic: find existing social account → return token; find user by email → attach social account → return token; neither → create new user → return token
- [ ] New OAuth users: `email_verified_at` set immediately (provider has already verified)
- [ ] `GET /api/v1/auth/social/{provider}/redirect` (provider: google, github, linkedin)
- [ ] `GET /api/v1/auth/social/{provider}/callback`
- [ ] Guard: unknown providers return HTTP 422
- [ ] Add `GOOGLE_CLIENT_ID`, `GITHUB_CLIENT_ID`, `LINKEDIN_CLIENT_ID` to `.env.example`
- [ ] Pest: new user via OAuth, returning user via OAuth, email collision → attach account

**Acceptance criteria:**
- OAuth users receive a Sanctum token without setting a password
- A user who registered via email can later link a Google account
- An unknown provider slug returns HTTP 422, not 500

---

### M1-05 — Role system and policies

**Branch:** `feature/m1-05-roles-policies`
**Labels:** `backend`, `security`, `backlog`
**Depends on:** M1-03

**Objective:**
Define role-based access control and all authorization policies.

**Tasks:**
- [ ] `UserRole` enum verified and bound correctly in `AuthServiceProvider`
- [ ] `UserPolicy` — `view`, `update`, `deactivate`, `viewSensitiveData`
- [ ] `EnforceSellerRole` middleware — returns HTTP 403 for non-sellers on seller routes
- [ ] `EnsureProfileComplete` middleware — returns HTTP 403 for sellers with incomplete profiles
- [ ] Register all policies in `AuthServiceProvider`
- [ ] Sanctum token abilities: `buyer`, `seller`, `admin` scopes defined
- [ ] Route groups in `api.php` with correct middleware stacks documented
- [ ] Pest: buyer hitting seller-only endpoint gets 403; admin can access any endpoint; policy `update` denies non-owner

**Acceptance criteria:**
- A buyer user hitting `POST /api/v1/services` gets HTTP 403
- A seller user hitting their own profile update gets HTTP 200
- A seller user hitting another seller's profile update gets HTTP 403

---

### M1-06 — Seller onboarding wizard

**Branch:** `feature/m1-06-seller-onboarding`
**Labels:** `backend`, `security`, `backlog`
**Depends on:** M1-05

**Objective:**
Five-step multi-session onboarding wizard. The `users` table is not updated
until the application is fully approved.

**Tasks:**
- [ ] `SellerOnboardingApplication` model with all JSON columns and enums
- [ ] `OnboardingStep` enum (orientation → personal_info → professional_info → phone_verification → under_review → approved → rejected)
- [ ] `OnboardingStatus` enum (in_progress, submitted, approved, rejected)
- [ ] `SellerOnboardingService` with all step methods and `assertStep()` guard
- [ ] Step DTOs: `OrientationStepDTO`, `PersonalInfoStepDTO`, `ProfessionalInfoStepDTO`, `PhoneVerificationDTO`
- [ ] `PhoneVerificationService` — send OTP via Vonage, verify OTP
- [ ] `SellerOnboardingController` with all 7 endpoints:
  - `POST /api/v1/seller-onboarding` (initiate — idempotent)
  - `GET /api/v1/seller-onboarding` (current state — for resume)
  - `POST /api/v1/seller-onboarding/orientation`
  - `POST /api/v1/seller-onboarding/personal`
  - `POST /api/v1/seller-onboarding/professional`
  - `POST /api/v1/seller-onboarding/phone/send` (throttle: 3/10min)
  - `POST /api/v1/seller-onboarding/phone/verify`
- [ ] Pest: full happy path, step-out-of-order rejection (HTTP 422), ToS not accepted, OTP rate limit (HTTP 429), idempotent initiation

**Acceptance criteria:**
- User can leave mid-flow and resume — `GET /api/v1/seller-onboarding` returns correct current step
- Attempting step 3 while on step 1 returns HTTP 422
- OTP send after 3 attempts in 10 minutes returns HTTP 429

---

### M1-07 — Onboarding approval pipeline

**Branch:** `feature/m1-07-onboarding-approval`
**Labels:** `backend`, `security`, `backlog`
**Depends on:** M1-06

**Objective:**
Automated fraud check and approval pipeline. The `users` table is only modified
inside a DB transaction at the moment of approval.

**Tasks:**
- [ ] `OnboardingFraudCheckService` with four signal detectors:
  - Phone linked to a suspended account
  - Copied profile content (similarity hash)
  - Flagged IP address
  - Suspicious completion speed (under 3 minutes)
- [ ] `FraudCheckResult` value object (signals array, isFlagged bool, riskScore 0–100)
- [ ] `SellerOnboardingApprovalPipeline` (Type 3 Service) — `review()`, `approve()`, `reject()`
- [ ] `RunSellerOnboardingReviewJob` (queued, high priority) — dispatched after phone verification
- [ ] `UpgradeToSellerAction` (Orchestrator) — inside DB transaction: updates user role, creates `seller_profiles` row, creates `seller_skills` pivot rows, creates `seller_languages` pivot rows
- [ ] `FraudSignal` model and writes
- [ ] Events: `SellerApplicationSubmittedNotificationEvent`, `SellerApplicationApprovedNotificationEvent`, `SellerApplicationRejectedNotificationEvent`
- [ ] Listeners: `SendApplicationReceivedEmailListener`, `SendSellerApprovalEmailListener`, `SendRejectionEmailListener` (all queued)
- [ ] Pest: auto-approve on clean application, auto-reject on profile score < 60, fraud signal triggers manual review queue, `users` table unchanged if transaction rolls back

**Acceptance criteria:**
- `users.role` is only updated inside `UpgradeToSellerAction`'s DB transaction
- If the transaction rolls back, `users.role` remains `buyer` and `seller_profiles` has no row
- Rejected application can be resubmitted — same application row updated, not a new row

---

### M1-08 — User profile CRUD

**Branch:** `feature/m1-08-user-profile`
**Labels:** `backend`, `backlog`
**Depends on:** M1-05

**Objective:**
Public profile reads with Redis caching and authenticated profile updates.

**Tasks:**
- [ ] `UserRepository` — `findWithSellerProfile()`, `paginateActiveSellers()` (only where needed — justify in PR)
- [ ] `UserProfileService` — `getPublicProfile()` (cached, tag: `user:{id}`, TTL 1h), `updateProfile()` (flushes cache tag)
- [ ] `UserResource` — public fields only
- [ ] `UserProfileResource` — includes sensitive fields for own profile
- [ ] `SellerProfileResource`
- [ ] `GET /api/v1/me` — own full profile (authenticated)
- [ ] `GET /api/v1/users/{user}` — public profile (cached)
- [ ] `PATCH /api/v1/users/{user}` — own profile update (authorized via UserPolicy)
- [ ] `POST /api/v1/me/avatar` — upload to S3, validate (max 2MB, jpeg/png/webp), return URL, update `users.avatar_url`
- [ ] Seller skills CRUD: `GET/POST/DELETE /api/v1/me/skills`
- [ ] Seller languages CRUD: `GET/POST/DELETE /api/v1/me/languages`
- [ ] Pest: public profile is cached on second request, PATCH by non-owner returns 403, avatar validates file type

**Acceptance criteria:**
- Second request to `GET /api/v1/users/{user}` hits Redis cache (verified via Redis monitor)
- Updating own profile flushes the cache (next request hits DB again)
- Avatar upload with a PDF file returns HTTP 422

---

### M1-09 — Taxonomy API

**Branch:** `feature/m1-09-taxonomy-api`
**Labels:** `backend`, `backlog`
**Depends on:** M1-02

**Objective:**
Read-only public API for all controlled vocabulary tables.
Cached aggressively — taxonomy changes very rarely.

**Tasks:**
- [ ] Category, Skill, Language, Country, Tag Eloquent models with relationships and scopes
- [ ] `TaxonomyService` — `getCategoryTree()` (cached 24h, tag: `categories`), `getSkillsTypeahead()` (cached 1h)
- [ ] `GET /api/v1/categories` — full tree (parent + children), cached 24h
- [ ] `GET /api/v1/categories/{category}/subcategories`
- [ ] `GET /api/v1/skills?q=` — typeahead, max 20 results, min 2 chars, cached 1h
- [ ] `GET /api/v1/skills?category_id=` — skills filtered by category
- [ ] `GET /api/v1/languages` — all active, cached 24h
- [ ] `GET /api/v1/countries` — all active with region grouping, cached 24h
- [ ] `GET /api/v1/tags?q=` — tag typeahead for service creation
- [ ] `CategoryResource`, `SkillResource`, `LanguageResource`, `CountryResource`
- [ ] Pest: category tree has correct parent-child structure, skill typeahead returns max 20, cache warms on first request

**Acceptance criteria:**
- Category tree is returned in a single query (no N+1 — verified with query log)
- Skill typeahead with `q=php` returns skills containing "php" in the name
- A second request to `/api/v1/categories` is served from Redis

---

### M1-10 — Service listing CRUD and packages

**Branch:** `feature/m1-10-service-listings`
**Labels:** `backend`, `backlog`
**Depends on:** M1-05, M1-09

**Objective:**
Sellers create and manage service listings. Each listing has packages, extras,
media, FAQs, requirements, and skills attached.

**Tasks:**

*Core listing CRUD:*
- [ ] `MarketplaceService` model with `ServiceStatus` enum and all relationships
- [ ] `ServicePolicy` — create (is seller), update/delete (owns service), view (published OR owner)
- [ ] `CreateServiceAction` (Orchestrator — from M0-08, expanded)
- [ ] `PublishServiceAction` (Orchestrator — validates completeness before publishing)
- [ ] `PauseServiceAction` / `UnpauseServiceAction` (Orchestrators)
- [ ] `POST /api/v1/services` — create (seller only, returns HTTP 201)
- [ ] `GET /api/v1/services/{service}` — public detail (published only for non-owners)
- [ ] `PATCH /api/v1/services/{service}` — update (owner only)
- [ ] `DELETE /api/v1/services/{service}` — soft delete (cannot delete if active orders)
- [ ] `POST /api/v1/services/{service}/publish`
- [ ] `POST /api/v1/services/{service}/pause`
- [ ] `GET /api/v1/me/services` — seller's own listings (filterable by status)

*Child resources:*
- [ ] `ServicePackage` model — max 3 packages per service (basic/standard/premium)
- [ ] `POST/PATCH/DELETE /api/v1/services/{service}/packages`
- [ ] `ServiceExtra` model
- [ ] `POST/PATCH/DELETE /api/v1/services/{service}/extras`
- [ ] `ServiceMedia` model — S3 upload, signed URL generation
- [ ] `POST /api/v1/services/{service}/media` — validate (image: max 5MB; video: max 75s, max 50MB)
- [ ] `DELETE /api/v1/services/{service}/media/{media}`
- [ ] `ServiceFaq` model with sort_order
- [ ] `POST/PATCH/DELETE /api/v1/services/{service}/faqs`
- [ ] `ServiceRequirement` model (type: text/multiple_choice/file)
- [ ] `POST/PATCH/DELETE /api/v1/services/{service}/requirements`
- [ ] Service skills pivot: `POST /api/v1/services/{service}/skills` (sync, max 10 skills)
- [ ] Service tags pivot: `POST /api/v1/services/{service}/tags` (max 5 tags)
- [ ] `ServiceResource`, `ServicePackageResource`, `ServiceCollection`
- [ ] Events: `ServicePublishedNotificationEvent` → `IndexServiceInSearchListener` (queued, low priority)
- [ ] Pest: create, publish (requires package), pause, delete policy, media type validation, tag limit enforcement

**Acceptance criteria:**
- A service cannot be published without at least one package (returns HTTP 422)
- A service cannot have more than 5 tags (returns HTTP 422)
- Soft-deleting a service with active orders returns HTTP 409
- `ServicePublishedNotificationEvent` is dispatched when a service is published

---

---

# Milestone 2 — Orders + Messaging + Reviews

**Gate condition:** All M1 issues closed.

---

### M2-01 — Buyer requests and custom offers

**Branch:** `feature/m2-01-buyer-requests`
**Labels:** `backend`, `backlog`
**Depends on:** M1-10

**Objective:**
Buyers post briefs. Sellers respond with custom offers. Accepted offer creates an order.

**Tasks:**
- [ ] `BuyerRequest` model with `BuyerRequestStatus` enum
- [ ] `CustomOffer` model with `CustomOfferStatus` enum
- [ ] `BuyerRequestPolicy`, `CustomOfferPolicy`
- [ ] `POST /api/v1/buyer-requests` — buyer creates brief
- [ ] `GET /api/v1/buyer-requests` — sellers browse open requests (filterable by category)
- [ ] `GET /api/v1/buyer-requests/{request}`
- [ ] `PATCH /api/v1/buyer-requests/{request}/close`
- [ ] `POST /api/v1/conversations/{conversation}/offers` — seller sends offer within thread
- [ ] `GET /api/v1/me/offers/sent`
- [ ] `GET /api/v1/me/offers/received`
- [ ] `PATCH /api/v1/offers/{offer}/accept` — triggers `PlaceOrderAction` automatically
- [ ] `PATCH /api/v1/offers/{offer}/decline`
- [ ] `PATCH /api/v1/offers/{offer}/withdraw`
- [ ] `ExpireCustomOffersJob` — scheduled daily, expires offers past `expires_at`
- [ ] Pest: accept creates order, buyer cannot send offer (403), withdrawn offer cannot be accepted

**Acceptance criteria:**
- Accepting an offer creates an Order record in one DB transaction
- A buyer cannot call `POST /api/v1/conversations/{conversation}/offers` (HTTP 403)
- An expired offer returns HTTP 409 on accept attempt

---

### M2-02 — Order placement and requirements flow

**Branch:** `feature/m2-02-order-placement`
**Labels:** `backend`, `backlog`
**Depends on:** M1-10

**Objective:**
Buyers place orders, pay upfront, and submit requirements.
Order does not start until requirements are submitted.

**Tasks:**
- [ ] `Order` model expanded with `OrderStatus` enum and all relationships
- [ ] `PlaceOrderAction` (Orchestrator — from M0-08 reference, expanded with real pricing and Stripe)
- [ ] `PlaceOrderDTO` with `fromRequest()` factory
- [ ] `SubmitRequirementsAction` (Orchestrator) — validates requirements, transitions to `in_progress`, sets `due_at`
- [ ] `OrderPolicy` — buyer and seller of that specific order can view; buyer submits requirements
- [ ] `POST /api/v1/orders` — buyer places order (requires verified email)
- [ ] `GET /api/v1/orders/{order}`
- [ ] `GET /api/v1/me/orders/buying` — buyer's orders (filterable by status)
- [ ] `GET /api/v1/me/orders/selling` — seller's queue (filterable by status)
- [ ] `POST /api/v1/orders/{order}/requirements` — submit requirements
- [ ] `OrderResource`, `OrderCollection`
- [ ] Guard: buyer cannot order their own service (HTTP 422)
- [ ] Order price snapshotted from package at time of purchase — future package changes do not affect order
- [ ] Pest: place order, requirements submission transitions status, due_at calculation, own-service guard

**Acceptance criteria:**
- `orders.price` equals the package price at time of purchase — changing the package price does not affect the existing order
- Order stays `pending_requirements` until `POST /api/v1/orders/{order}/requirements` is called
- `due_at` is calculated as `requirements_submitted_at + package.delivery_days`

---

### M2-03 — Order lifecycle

**Branch:** `feature/m2-03-order-lifecycle`
**Labels:** `backend`, `backlog`
**Depends on:** M2-02

**Objective:**
All order state transitions from `in_progress` through to `completed`,
with auto-completion timer.

**Tasks:**
- [ ] `OrderTransitionService` (Domain Algorithm) — enforces valid state machine transitions, throws `InvalidOrderTransitionException` for illegal moves
- [ ] `SubmitDeliveryAction` (Orchestrator) — attaches delivery, transitions to `delivered`
- [ ] `RequestRevisionAction` (Orchestrator) — validates revision count, transitions back to `in_progress`
- [ ] `CompleteOrderAction` (Orchestrator) — transitions to `completed`, fires `OrderCompletedNotificationEvent`
- [ ] `CancelOrderAction` (Orchestrator) — handles mutual and admin-forced cancellation
- [ ] `POST /api/v1/orders/{order}/deliver`
- [ ] `POST /api/v1/orders/{order}/revisions`
- [ ] `POST /api/v1/orders/{order}/complete`
- [ ] `POST /api/v1/orders/{order}/cancel`
- [ ] `AutoCompleteOrderJob` — fires 3 days after delivery if buyer has not acted
- [ ] Listeners on `OrderCompletedNotificationEvent`: `ReleaseEscrowOnCompletionListener` (queued high), `PromptBuyerReviewListener` (queued default), `UpdateSellerStatsListener` (queued low)
- [ ] Pest: delivery → auto-complete after 3 days, revision count enforcement (cannot exceed package limit), transition from `completed` back to any state is rejected

**Acceptance criteria:**
- Buyer cannot request more revisions than `package.revision_count` allows
- `AutoCompleteOrderJob` fires exactly 3 days after `delivered_at` (verified via job dispatch assertion)
- Attempting an illegal state transition returns HTTP 409

---

### M2-04 — Resolution centre

**Branch:** `feature/m2-04-resolution-centre`
**Labels:** `backend`, `backlog`
**Depends on:** M2-03

**Objective:**
Structured dispute resolution flow. Buyers and sellers resolve problems without
admin involvement where possible. Admin escalation when needed.

**Tasks:**
- [ ] `OrderResolution` model with `ResolutionType` and `ResolutionStatus` enums
- [ ] `OrderResolutionService` (Type 3 Long-running Orchestration) — coordinates resolution outcomes
- [ ] `POST /api/v1/orders/{order}/resolutions` — open a resolution (one at a time enforced)
- [ ] `PATCH /api/v1/orders/{order}/resolutions/{resolution}/accept`
- [ ] `PATCH /api/v1/orders/{order}/resolutions/{resolution}/decline`
- [ ] `PATCH /api/v1/orders/{order}/resolutions/{resolution}/escalate` — flags for admin
- [ ] Delivery extension: updates `order.due_at` by requested days added to original (not current)
- [ ] Partial refund: creates Stripe refund, adjusts ledger entries, does NOT affect seller metrics
- [ ] Mutual cancellation: does NOT count against seller cancellation rate
- [ ] Admin-forced cancellation: DOES count against seller cancellation rate
- [ ] Guard: cannot open new resolution while one is already open — returns HTTP 409
- [ ] `ResolutionPolicy` — only buyer or seller of that order
- [ ] Pest: extension updates due_at correctly, partial refund flow, mutual cancel vs forced cancel metric difference

**Acceptance criteria:**
- `POST /api/v1/orders/{order}/resolutions` while one is open returns HTTP 409
- Mutual cancellation does not increment `seller_profiles.cancellation_count`
- Admin-forced cancellation does increment it
- Partial refund accepted → Stripe refund created in the same request (synchronous)

---

### M2-05 — Messaging

**Branch:** `feature/m2-05-messaging`
**Labels:** `backend`, `backlog`
**Depends on:** M1-03

**Objective:**
Full messaging system: pre-order inquiry threads and order-attached threads.
Real-time delivery via WebSockets.

**Tasks:**
- [ ] `Conversation` model with `ConversationType` enum (inquiry / order)
- [ ] `Message` model with attachments JSON
- [ ] `ConversationService` — `findOrCreateInquiry()`, `createOrderThread()` (called automatically when order is placed)
- [ ] `MessageDispatchAction` (Atomic) — creates message, broadcasts `MessageSentNotificationEvent`
- [ ] Install and configure Laravel Reverb
- [ ] `MessageSentNotificationEvent` broadcast on private channel `conversation.{id}`
- [ ] `GET /api/v1/conversations` — authenticated user's inbox sorted by `last_message_at`
- [ ] `POST /api/v1/conversations` — start new inquiry
- [ ] `GET /api/v1/conversations/{conversation}/messages` — paginated
- [ ] `POST /api/v1/conversations/{conversation}/messages` — send message
- [ ] `PATCH /api/v1/conversations/{conversation}/messages/{message}/read`
- [ ] `POST /api/v1/conversations/{conversation}/archive`
- [ ] File attachments: upload to S3 (max 10 files, max 20MB each)
- [ ] `ConversationPolicy`, `MessagePolicy`
- [ ] Spam guard: block messages from users with no order or request relationship
- [ ] Pest: message sent event dispatched, mark read, spam guard blocks unknown senders, order thread created on order placement

**Acceptance criteria:**
- Order placement automatically creates a conversation thread linked to the order
- Archived conversations do not appear in `GET /api/v1/conversations`
- `MessageSentNotificationEvent` is broadcast on the correct private channel

---

### M2-06 — Reviews — double-blind system

**Branch:** `feature/m2-06-reviews`
**Labels:** `backend`, `backlog`
**Depends on:** M2-03

**Objective:**
Double-blind review system. Neither party sees the other's review until both submit
or the 10-day window closes.

**Tasks:**
- [ ] `Review` model with `ReviewDirection` enum (buyer_to_seller / seller_to_buyer)
- [ ] `SubmitReviewAction` (Orchestrator) — stores review, `revealed_at` stays null
- [ ] `ReplyToReviewAction` (Orchestrator) — adds seller reply (once only, cannot edit)
- [ ] `ReviewRevealService` (Domain Algorithm — or Type 3 orchestration) — checks if both sides submitted; if yes sets `revealed_at` on both; if window expired sets `revealed_at` on submitted ones
- [ ] `RevealReviewsJob` — checks each unrevealed review after 10-day window
- [ ] `RatingAggregatorService` (Domain Algorithm) — recalculates `avg_rating`, `total_reviews` on `seller_profiles`
- [ ] `POST /api/v1/orders/{order}/reviews` — submit (only after order completed, within 10 days)
- [ ] `PATCH /api/v1/reviews/{review}/reply` — seller reply (once only)
- [ ] `GET /api/v1/users/{user}/reviews` — public list of revealed reviews only
- [ ] After reveal: dispatch `RecalculateSellerRatingJob` (queued, low priority)
- [ ] After rating recalculated: dispatch `IndexServiceInSearchJob` for all seller's published services
- [ ] `ReviewPolicy`
- [ ] Pest: buyer cannot see seller's review before reveal, reveal on both-submitted, reveal on window-expiry, second reply attempt returns 409

**Acceptance criteria:**
- `GET /api/v1/users/{user}/reviews` returns only reviews where `revealed_at` is not null
- A seller adding a second reply to the same review returns HTTP 409
- Rating recalculation fires within the same request cycle as review reveal (via dispatched job)

---

### M2-07 — Feature flags

**Branch:** `feature/m2-07-feature-flags`
**Labels:** `backend`, `backlog`
**Depends on:** M1-01

**Objective:**
Runtime feature toggles managed from the admin panel.
Used to gate seller onboarding beta, A/B search ranking, and emergency disables.

**Tasks:**
- [ ] Install Laravel Pennant: `composer require laravel/pennant`
- [ ] Configure Pennant to use `feature_flags` database table
- [ ] `FeatureFlagService` wrapper — `isEnabled(string $flag, ?User $user = null): bool`
- [ ] Seed: default flags with all features enabled (safe default)
- [ ] Verify feature flags can be checked in any Action without coupling to the HTTP layer
- [ ] Document the flag naming convention: `snake_case`, prefixed by domain (e.g., `seller_onboarding_beta`, `search_pro_filter`)
- [ ] Pest: flag enabled returns true, flag disabled returns false, flag scoped to specific user

**Acceptance criteria:**
- `FeatureFlagService::isEnabled('seller_onboarding_beta')` returns correct value from DB
- Disabling a flag in the DB takes effect without a deployment
- Scoped flag: flag disabled globally but enabled for a specific user returns true for that user

---

### M2-08 — Seller level calculation

**Branch:** `feature/m2-08-seller-levels`
**Labels:** `backend`, `backlog`
**Depends on:** M2-03

**Objective:**
Seller level is recalculated after every completed order.
The result is stored on `users.seller_level` for fast read performance.

**Tasks:**
- [ ] `SellerLevelCalculator` (Domain Algorithm) — evaluates KPIs against thresholds
  - New Seller: 0 orders
  - Level One: 10+ orders, 4.0+ rating, 90%+ completion, 90%+ on-time
  - Level Two: 50+ orders, 4.5+ rating, 95%+ completion, 95%+ on-time
  - Top Rated: manual review + all Level Two criteria
- [ ] `RecalculateSellerStatsJob` (queued, low priority) — triggered by `OrderCompletedNotificationEvent` listener
  - Updates all KPI fields on `seller_profiles`
  - Calls `SellerLevelCalculator::calculate()`
  - If level changed: updates `users.seller_level`, fires `SellerLevelChangedNotificationEvent`
- [ ] `SellerLevelChangedNotificationEvent` listener: `ReindexSellerServicesListener` (queued, low)
- [ ] Pest: level-up threshold test, level-down test, no change when thresholds not crossed, Top Rated is never auto-assigned

**Acceptance criteria:**
- A seller completing their 10th order with qualifying KPIs gets `seller_level` updated to `level_one`
- `users.seller_level` is always computed from KPIs — never set directly except by the calculator
- Top Rated is never assigned automatically — the calculator skips it

---

### M2-09 — Observability setup

**Branch:** `chore/m2-09-observability`
**Labels:** `devops`, `backend`, `backlog`
**Depends on:** M1-01

**Objective:**
Sentry error tracking, structured logging conventions, and Laravel Telescope
for local development. These must be in place before payment work begins.

**Tasks:**
- [ ] Install Sentry Laravel SDK: `composer require sentry/sentry-laravel`
- [ ] Configure Sentry in `.env.example` and `config/sentry.php`
- [ ] Verify unhandled exceptions are captured automatically in staging
- [ ] Configure Laravel Telescope for `local` and `staging` environments only (never production)
- [ ] Slow query logging: add `DB::whenQueryingForLongerThan(100, fn() => Log::warning(...))` to `AppServiceProvider`
- [ ] Document structured logging convention in `CONTRIBUTING.md`:
  - All log messages use dot notation: `domain.action` (e.g., `order.placing`)
  - All log entries include entity IDs as structured context (not in the message string)
  - No PII in any log entry
- [ ] Add `Job::failed()` method template to `CONTRIBUTING.md` — every job must implement it
- [ ] Pest: deliberate exception is captured by Sentry in test environment (mock Sentry client)

**Acceptance criteria:**
- A thrown exception in a controller is captured by Sentry (verified in Sentry dashboard)
- A query taking over 100ms triggers a warning log entry
- Telescope is accessible at `/telescope` in local environment only

---

---

# Milestone 3 — Payments + Billing

**Gate condition:** All M2 issues closed AND Architecture Steward confirms
zero open `architecture-exception` issues. No exceptions permitted from this milestone onward.

**Labels:** `backend`, `payment-critical`, `backlog`

---

### M3-01 — Stripe integration and webhooks

**Branch:** `feature/m3-01-stripe-integration`
**Labels:** `backend`, `payment-critical`, `backlog`
**Depends on:** M2-02

**Objective:**
Stripe PaymentIntent creation for order placement and webhook handling.
All financial writes are synchronous inside DB transactions.

**Tasks:**
- [ ] Install Stripe PHP SDK: `composer require stripe/stripe-php`
- [ ] `config/payment.php` — platform_fee_percent (20), buyer_fee_percent (5.5), small_order_threshold (10000 cents), small_order_surcharge (300 cents), standard_clearing_days (14), top_rated_clearing_days (7)
- [ ] `FeeCalculator` (Domain Algorithm) — `calculate(int $priceInCents): FeeBreakdown`
  - Returns: buyer_total, platform_fee, buyer_fee, seller_net
  - All values in integer cents — no float arithmetic
  - Unit tested for correctness at boundary values
- [ ] `StripePaymentGateway` (Type 2 — in `app/Infrastructure/`) — implements `PaymentGatewayInterface`
  - `createPaymentIntent(int $amount, string $currency, array $metadata): string`
  - `refund(string $paymentIntentId, ?int $amount = null): void`
  - `createConnectAccount(array $userDetails): string`
  - `transferToConnect(string $connectAccountId, int $amount): string`
- [ ] `ValidateStripeWebhook` middleware — verifies `Stripe-Signature` header, returns HTTP 400 on invalid
- [ ] `WebhookController::handle()` — unauthenticated route, protected by middleware
  - `payment_intent.succeeded` → `PaymentCapturedStateEvent` (synchronous, 1 listener)
  - `payment_intent.payment_failed` → cancels order, notifies buyer
  - `charge.refunded` → updates payment record, writes ledger
  - `transfer.paid` → updates withdrawal to completed, writes ledger debit
  - `transfer.failed` → updates withdrawal to failed, reverses ledger credit, notifies seller
- [ ] All webhook handlers are idempotent — safe to receive same event twice
- [ ] `POST /api/v1/payments/webhook` (no auth middleware)
- [ ] Pest: fee calculation at boundary values, invalid webhook signature returns 400, duplicate event is idempotent

**Acceptance criteria:**
- `FeeCalculator::calculate(10000)` returns correct integer values — no floating-point leakage
- An invalid Stripe signature on the webhook endpoint returns HTTP 400, not 500
- Receiving `payment_intent.succeeded` twice does not credit the seller twice

---

### M3-02 — Escrow and clearing pipeline

**Branch:** `feature/m3-02-escrow-clearing`
**Labels:** `backend`, `payment-critical`, `backlog`
**Depends on:** M3-01

**Objective:**
Hold buyer payment in escrow. Release after order completion. Clear after
the clearing period. All writes are synchronous inside DB transactions.

**Tasks:**
- [ ] `Payment` model with `PaymentStatus` enum
- [ ] `EscrowEntry` model with `EscrowStatus` enum
- [ ] `LedgerEntry` model — immutable, no `updated_at`, no soft deletes
- [ ] `PaymentEscrowService` — `hold()`: creates Payment record, creates EscrowEntry with `clearance_at` (computed from seller's level), writes LedgerEntry (debit buyer)
- [ ] `PaymentCapturedStateEvent` listener: `HoldPaymentInEscrowAction` (Atomic — synchronous, inside the webhook transaction)
- [ ] `ReleaseEscrowJob` — dispatched by `OrderCompletedNotificationEvent` listener; transitions escrow to clearing, writes LedgerEntry (pending credit)
- [ ] `ProcessClearingJob` — scheduled every hour via `app/Console/Kernel.php`; finds escrows where `clearance_at <= now()` and status = `clearing`; transitions to `available`; writes LedgerEntry (credit seller); idempotent
- [ ] Seller balance computed from ledger: `GET /api/v1/me/balance` returns `{available, pending_clearance, lifetime_earned}`
- [ ] `RefundProcessingService` — full refund and partial refund; writes LedgerEntry on each
- [ ] Pest: escrow created on payment, clearing fires after correct days (14 vs 7 for Top Rated), `ProcessClearingJob` idempotent, partial refund reduces escrow correctly, seller balance equals sum of ledger entries

**Acceptance criteria:**
- Top Rated seller's `clearance_at` is 7 days after completion — all others are 14 days
- Running `ProcessClearingJob` twice does not credit the seller twice
- `GET /api/v1/me/balance` returns values that match the sum of `ledger_entries` for that user

---

### M3-03 — Seller payouts

**Branch:** `feature/m3-03-seller-payouts`
**Labels:** `backend`, `payment-critical`, `backlog`
**Depends on:** M3-02

**Objective:**
Sellers withdraw cleared earnings. Daily limit, method-change lockout,
and Stripe Connect transfer.

**Tasks:**
- [ ] Stripe Connect account creation: `CreateStripeConnectAccountAction` (Atomic) — triggered by `SellerApplicationApprovedNotificationEvent` listener
- [ ] `Withdrawal` model with `WithdrawalStatus` and `WithdrawalMethod` enums
- [ ] `SellerPayoutService` — `requestWithdrawal()`:
  - Validates amount ≤ available balance
  - Enforces $5,000/day limit (sum of today's withdrawals + requested amount)
  - Enforces 24h method-change lockout
  - Creates Withdrawal record with status `pending`
  - Initiates Stripe Connect transfer
  - Writes LedgerEntry (debit)
- [ ] `POST /api/v1/me/withdrawals` — request payout (minimum $20)
- [ ] `GET /api/v1/me/withdrawals` — withdrawal history
- [ ] `PATCH /api/v1/me/withdrawal-method` — update payout method (triggers 24h lockout)
- [ ] Stripe `transfer.paid` webhook: transitions Withdrawal to `completed`
- [ ] Stripe `transfer.failed` webhook: transitions to `failed`, reverses LedgerEntry, notifies seller
- [ ] Pest: over-balance attempt returns 422, daily limit enforced, method-change lockout, failed transfer reverses ledger

**Acceptance criteria:**
- Withdrawal request exceeding available balance returns HTTP 422
- Two withdrawals in 24h that together exceed $5,000 — the second returns HTTP 422
- A failed Stripe transfer writes a reversing credit to `ledger_entries` within the same webhook handler

---

### M3-04 — Billing — fee rules and invoices

**Branch:** `feature/m3-04-billing`
**Labels:** `backend`, `backlog`
**Depends on:** M3-01

**Objective:**
Invoice generation for completed orders. Tax handling for EU/UK buyers.
Separated from the Payment domain by design.

**Tasks:**
- [ ] `Invoice` model
- [ ] `InvoiceDTO`
- [ ] `InvoiceGenerationService` (Type 3 Orchestration) — generates PDF receipt after order completes; stores in S3
- [ ] `TaxCalculationService` (Domain Algorithm) — calculates VAT/GST by buyer country using `countries` table; returns zero for non-taxable countries
- [ ] Invoice PDF generation via `barryvdh/laravel-dompdf`
- [ ] `GET /api/v1/orders/{order}/invoice` — returns S3 URL for the generated PDF
- [ ] Invoice auto-generated and emailed on `OrderCompletedNotificationEvent` (queued listener)
- [ ] Pest: invoice generated after order completion, EU buyer has VAT applied, non-EU buyer has zero tax

**Acceptance criteria:**
- Invoice PDF is accessible at the S3 URL returned by `GET /api/v1/orders/{order}/invoice`
- A buyer with country code `SE` (Sweden) has 25% VAT applied on the invoice
- A buyer with country code `US` has zero tax on the invoice

---

### M3-05 — Platform credits

**Branch:** `feature/m3-05-platform-credits`
**Labels:** `backend`, `backlog`
**Depends on:** M3-02

**Objective:**
Admin-issued platform credits that buyers can use toward purchases.
Same fee structure as regular payments.

**Tasks:**
- [ ] `PlatformCredit` model with `used_at` and `expires_at`
- [ ] `IssuePlatformCreditAction` (Atomic) — admin-only, writes credit record and LedgerEntry
- [ ] Credit redemption integrated into `PlaceOrderAction` — if buyer has available credits, apply before charging Stripe
- [ ] Credits carry the same buyer fee (5.5%) — no fee advantage
- [ ] `GET /api/v1/me/credits` — buyer's available credits and history
- [ ] Pest: credit applied to order reduces Stripe charge amount, expired credit cannot be used, fee still applied on credit amount

**Acceptance criteria:**
- An order placed with enough credits to cover the full amount charges Stripe $0
- An expired credit is ignored in the credit-application logic
- The buyer fee is applied to the credit amount, not waived

---

### M3-06 — Financial audit and ledger verification

**Branch:** `chore/m3-06-ledger-audit`
**Labels:** `backend`, `payment-critical`, `backlog`
**Depends on:** M3-03, M3-04

**Objective:**
Verify the ledger is consistent across all financial flows. Add an Artisan
command for financial reconciliation. This is a quality gate before M4.

**Tasks:**
- [ ] `php artisan ledger:reconcile` Artisan command:
  - For each user: sum of all credit entries minus sum of all debit entries = current available balance
  - Report any discrepancies to `Log::error()` with full context
  - Exits with non-zero code if any discrepancy found
- [ ] Verify reconcile command passes against all M3 test data
- [ ] Document reconciliation schedule (recommended: run daily in production)
- [ ] Add `make reconcile` to `Makefile`
- [ ] Pest: command exits 0 on balanced ledger, exits 1 on discrepancy, discrepancy logged with correct context

**Acceptance criteria:**
- `php artisan ledger:reconcile` passes on a seeded database with simulated orders and payouts
- Manually inserting an orphaned ledger entry causes the command to exit non-zero and log the discrepancy
- Command completes in under 30 seconds on a dataset of 10,000 orders

---

### M3-07 — Stripe webhook reliability

**Branch:** `chore/m3-07-webhook-reliability`
**Labels:** `backend`, `payment-critical`, `devops`, `backlog`
**Depends on:** M3-01

**Objective:**
Webhooks are the most failure-prone integration surface. This issue adds
idempotency keys, dead-letter handling, and monitoring for webhook failures.

**Tasks:**
- [ ] `stripe_webhook_events` table — stores `event_id`, `type`, `processed_at`, `status`
- [ ] `WebhookController` checks for duplicate `event_id` before processing — skips if already processed
- [ ] Failed webhook processing routes to dead-letter queue (separate Redis queue)
- [ ] `php artisan webhooks:retry-failed` — retries all items in the dead-letter queue
- [ ] Filament widget: displays count of failed/pending webhook events (added in M6)
- [ ] Alert: if dead-letter queue grows beyond 10 items, `Log::critical()` is triggered
- [ ] Pest: duplicate event is skipped, failed processing enters dead-letter queue, retry command processes it

**Acceptance criteria:**
- Sending the same Stripe event ID twice results in exactly one `Payment` record created
- A simulated processing failure puts the event in the dead-letter queue
- `php artisan webhooks:retry-failed` successfully processes the failed event

---

---

# Milestone 4 — Notifications + Analytics

**Gate condition:** All M3 issues closed.

---

### M4-01 — Notification dispatch system

**Branch:** `feature/m4-01-notifications`
**Labels:** `backend`, `backlog`
**Depends on:** M1-03

**Objective:**
Centralised multi-channel notification dispatch based on user preferences.

**Tasks:**
- [ ] `NotificationDispatchService` — reads `notification_preferences` before dispatching to each channel
- [ ] `UserDevice` model for push token management
- [ ] `POST /api/v1/me/devices` — register push token
- [ ] `DELETE /api/v1/me/devices/{token}` — unregister
- [ ] `GET /api/v1/me/notifications` — paginated (unread first)
- [ ] `PATCH /api/v1/me/notifications/{id}/read`
- [ ] `PATCH /api/v1/me/notifications/read-all`
- [ ] `GET /api/v1/me/notification-preferences`
- [ ] `PATCH /api/v1/me/notification-preferences`
- [ ] Implement all 12 Notification classes: `OrderPlaced`, `OrderDelivered`, `OrderCompleted`, `RevisionRequested`, `PaymentReceived`, `ReviewSubmitted`, `SellerApproved`, `SellerRejected`, `DisputeOpened`, `DisputeResolved`, `ClearingComplete`, `SellerLevelChanged`
- [ ] Channels: database (in-app bell), mail (Mailgun/SES), push (FCM + APNs), SMS (Vonage — critical only)
- [ ] Real-time bell count update: broadcast `NewNotificationBroadcast` via Reverb on database channel save
- [ ] All queued notification listeners declare `$afterCommit = true`
- [ ] Pest: user with email disabled still gets in-app notification, no double-dispatch on idempotency check

**Acceptance criteria:**
- A user who disables email for `order_placed` still receives the in-app bell notification
- The notification bell count updates in the browser without a page refresh (Reverb broadcast verified)
- The same event cannot trigger the same notification twice (idempotency check in `NotificationDispatchService`)

---

### M4-02 — Seller analytics

**Branch:** `feature/m4-02-seller-analytics`
**Labels:** `backend`, `backlog`
**Depends on:** M2-08, M3-02

**Objective:**
Seller dashboard analytics endpoints. All data computed from existing tables —
no separate analytics database at this scale.

**Tasks:**
- [ ] `SellerAnalyticsService` — aggregates data from `orders`, `ledger_entries`, `reviews`
- [ ] `GET /api/v1/me/analytics/overview` — earnings (all-time, this month), orders (total, active, completed, cancelled), rates (completion, on-time, response), avg_rating, unique_buyers, repeat_buyer_percentage
- [ ] `GET /api/v1/me/analytics/earnings?period=month|year` — earnings breakdown with chart data points
- [ ] `GET /api/v1/me/analytics/cancellations` — breakdown by cancellation type (last 60 days)
- [ ] `GET /api/v1/me/analytics/buyers/geography` — country breakdown as percentage of total sales
- [ ] All analytics queries complete in under 100ms (verified with query log in tests)
- [ ] Pest: overview returns correct totals, geography returns percentage values summing to 100, empty seller returns zeroes not errors

**Acceptance criteria:**
- A seller with zero orders receives a valid response with all zeroes — not a 500 error
- All analytics queries hit the correct indexes (verified — no full table scans on large datasets)
- `earnings` monthly breakdown data points match the sum of `ledger_entries` for that month

---

### M4-03 — Queue worker configuration

**Branch:** `chore/m4-03-queue-workers`
**Labels:** `devops`, `backlog`
**Depends on:** M1-01

**Objective:**
Configure Laravel Horizon with named queues, priorities, and worker pools.

**Tasks:**
- [ ] Install Laravel Horizon: `composer require laravel/horizon`
- [ ] `config/horizon.php` with five named queue workers:
  - `critical` (1 worker, no timeout) — payment webhooks, fraud flags, account bans
  - `high` (2 workers) — order notifications, OTP, onboarding review
  - `default` (3 workers) — general application jobs
  - `low` (2 workers) — search indexing, rating recalculation, seller stats
  - `batch` (1 worker) — email digests, report generation, full reindex
- [ ] All existing jobs in codebase have explicit queue assignments matching the priority table
- [ ] Horizon dashboard accessible at `/horizon` (admin only — guarded by `HorizonServiceProvider`)
- [ ] Verify `php artisan horizon` starts all workers without errors
- [ ] Add `make horizon` to `Makefile`

**Acceptance criteria:**
- A `PaymentCapturedStateEvent` listener's job runs on the `critical` queue (verified in Horizon)
- A search indexing job runs on the `low` queue (verified in Horizon)
- No job uses the `default` queue without a justifying comment

---

### M4-04 — Email templates

**Branch:** `feature/m4-04-email-templates`
**Labels:** `frontend`, `backend`, `backlog`
**Depends on:** M4-01

**Objective:**
Branded HTML email templates for all transactional notification types.

**Tasks:**
- [ ] Create base email layout: `resources/views/emails/layout.blade.php`
  - Platform logo, consistent typography, footer with unsubscribe link
- [ ] Create individual Mailable classes for all 12 notification types
- [ ] Each template includes: relevant context (order ID, amount, service name), clear CTA button, plain-text fallback
- [ ] Test all templates in Mailtrap (staging) — verify rendering in Gmail, Outlook, Apple Mail
- [ ] Configure SES or Mailgun in `.env.example`
- [ ] Pest: each Mailable contains the expected subject line and contains the correct data in the body

**Acceptance criteria:**
- All 12 email templates render without errors in Mailtrap
- Every email has a plain-text fallback (verified by checking the `text()` method on each Mailable)
- The unsubscribe link in the footer routes to the notification preferences endpoint

---

### M4-05 — Scheduled jobs and maintenance tasks

**Branch:** `chore/m4-05-scheduled-jobs`
**Labels:** `backend`, `devops`, `backlog`
**Depends on:** M3-02, M2-01

**Objective:**
Register all scheduled jobs in the console kernel and verify they run correctly.

**Tasks:**
- [ ] `app/Console/Kernel.php` — register all scheduled commands:
  - `ProcessClearingJob` — every hour
  - `ExpireCustomOffersJob` — daily at midnight
  - `RecalculateSellerStatsJob` (batch run for missed completions) — daily at 2am
  - `php artisan horizon:snapshot` — every 5 minutes (Horizon metrics)
  - `php artisan ledger:reconcile` — daily at 3am (logs discrepancies, does not fail silently)
  - Telescope pruning — daily (local/staging only)
- [ ] Verify all scheduled commands are idempotent
- [ ] Add `make schedule` to `Makefile` for local testing: `php artisan schedule:run`
- [ ] Pest: each scheduled command can be dispatched manually and completes without errors

**Acceptance criteria:**
- `php artisan schedule:run` completes without errors on a seeded database
- All scheduled commands are idempotent — running them twice produces the same result as running once

---

---

# Milestone 5 — Search

**Gate condition:** M1 complete. (Search can be built in parallel with M2–M4 after M1.)

---

### M5-01 — Elasticsearch index setup

**Branch:** `feature/m5-01-elasticsearch-setup`
**Labels:** `backend`, `devops`, `backlog`
**Depends on:** M1-01

**Objective:**
Create and configure the Elasticsearch index. Establish the versioning strategy
before any documents are indexed.

**Tasks:**
- [ ] Install `elasticsearch/elasticsearch-php` (^8.0)
- [ ] `config/elasticsearch.php` — host, port, index alias (`service_listings`), timeout settings
- [ ] `ElasticsearchClient` (Type 2 Infrastructure) — implements `SearchIndexInterface`
- [ ] `ServiceListingMappingV1` — full index mapping with correct field types (text+keyword, keyword, integer, float, boolean, date)
- [ ] English analyzer with stopwords in mapping settings
- [ ] `php artisan search:create-index {version}` — creates versioned index
- [ ] `php artisan search:swap-alias {version}` — points alias to version
- [ ] `php artisan search:reindex {from} {to}` — copies documents between versions
- [ ] `php artisan search:delete-index {version}` — safe deletion
- [ ] `php artisan search:reindex-all` — full rebuild from MySQL
- [ ] Elasticsearch 8 added to Sail `docker-compose.yml` (already in M0-01 — verify it is running)
- [ ] Pest: index can be created, document can be indexed, document can be retrieved by ID

**Acceptance criteria:**
- `php artisan search:create-index v1` creates `service_listings_v1` with correct mapping
- `php artisan search:swap-alias v1` makes `service_listings` alias point to `service_listings_v1`
- `php artisan search:reindex-all` indexes all published services from MySQL

---

### M5-02 — Search sync pipeline

**Branch:** `feature/m5-02-search-sync`
**Labels:** `backend`, `backlog`
**Depends on:** M5-01, M1-10

**Objective:**
Keep Elasticsearch in sync with MySQL. Every relevant data change triggers
a reindex of the affected service documents.

**Tasks:**
- [ ] `ServiceSearchDocument` DTO — all denormalised fields for one ES document
- [ ] `ServiceIndexBuilder` — builds document from eagerly loaded `MarketplaceService` with all relations
- [ ] `IndexServiceInSearchJob` (queued, low priority) — idempotent: removes document if service is not published
- [ ] `RemoveServiceFromSearchJob` (queued, low priority)
- [ ] Event listeners that dispatch `IndexServiceInSearchJob`:
  - `ServicePublishedNotificationEvent`
  - `ServiceUpdatedNotificationEvent`
  - `SellerProfileUpdatedNotificationEvent` → dispatch for all seller's published services
  - `SellerSkillsUpdatedNotificationEvent` → same
  - `SellerLanguagesUpdatedNotificationEvent` → same
  - `SellerLevelChangedNotificationEvent` → same
  - `ReviewsRevealedNotificationEvent` → `UpdateServiceRatingSignalsJob` (partial update)
- [ ] Dead-letter queue for failed index jobs — retry command in M5-01 Artisan commands
- [ ] Pest: publishing a service dispatches index job, updating seller country dispatches reindex for all services, removing service removes ES document

**Acceptance criteria:**
- Publishing a service results in an ES document within 30 seconds (via queued job)
- Updating a seller's country code updates all their service documents
- Deleting a service removes its ES document (verified by `search:reindex-all` followed by assertion)

---

### M5-03 — Search API

**Branch:** `feature/m5-03-search-api`
**Labels:** `backend`, `backlog`
**Depends on:** M5-02

**Objective:**
Public search endpoint with all filters, sort options, and facet aggregations.
MySQL is never queried during a search request.

**Tasks:**
- [ ] `ServiceSearchDTO` — all filter parameters with validation rules
- [ ] `ServiceSearchService` — builds full Elasticsearch bool query with `function_score`
- [ ] All filter implementations:
  - keyword (`multi_match` on title^3, tags^2, description — fuzziness AUTO)
  - category / subcategory (`term`)
  - skills — matches `skillSlugs` OR `sellerSkillSlugs` (either qualifies)
  - seller level (`terms`, multi-select)
  - country (`terms`, multi-select)
  - region (`terms`, multi-select)
  - language (`terms`, multi-select)
  - budget min/max (`range` on `priceMin`)
  - delivery days (`range` on `deliveryDaysMin`)
  - online only (`term`)
  - pro only (`term`)
- [ ] Sort modes: relevance (`_score` desc), best_selling (`totalOrders` desc + `avgRating` desc), newest (`publishedAt` desc), top_rated (`avgRating` desc + `totalOrders` desc), price_asc, price_desc
- [ ] `function_score` boosts: scoreBoost×1.5, avgRating×0.4, totalOrders (log1p)×0.1, completionRate×0.3, isOnline weight 1.1, hasVideo weight 1.05
- [ ] Facet aggregations in response: skills (top 50), seller_levels, countries (top 100), languages (top 50), price_stats, delivery_days distribution
- [ ] `GET /api/v1/search/services` — public, no auth required
- [ ] `SearchServicesRequest` FormRequest
- [ ] `SearchResultResource` — hits + total + facets + took(ms)
- [ ] Pest: keyword search returns relevant results, each filter type narrows results correctly, sort modes change order, empty results return 200 not 404

**Acceptance criteria:**
- Search responds in under 200ms on warm Elasticsearch (verified in test environment)
- Facet counts are correct and sum to the correct total for that filter dimension
- Empty query with no filters returns all published services sorted by relevance

---

### M5-04 — Search query logging

**Branch:** `feature/m5-04-search-logging`
**Labels:** `backend`, `backlog`
**Depends on:** M5-03

**Objective:**
Log every search query for ranking improvement and taxonomy gap detection.

**Tasks:**
- [ ] `SearchQuery` model mapped to `search_queries` table
- [ ] `LogSearchQueryJob` (queued, batch priority) — dispatched after every `GET /api/v1/search/services` response
- [ ] Logs: `query_text`, `filters_applied` (JSON), `result_count`, `user_id` (nullable — anonymised for guests), `took_ms`, `created_at`
- [ ] No PII stored — guest searches store a session hash, not IP address
- [ ] `GET /api/v1/admin/search/popular-queries` — Filament-accessible analytics (added in M6)
- [ ] Pest: search logs a `SearchQuery` record, zero-result queries are flagged for taxonomy review

**Acceptance criteria:**
- Every search request creates a `search_queries` row (verified in feature test)
- The `user_id` column is null for unauthenticated requests
- A search with `result_count = 0` has a nullable `flagged_at` column set — indicating a taxonomy gap

---

### M5-05 — Autocomplete

**Branch:** `feature/m5-05-autocomplete`
**Labels:** `backend`, `backlog`
**Depends on:** M5-01

**Objective:**
Fast typeahead suggestions for the search bar. Separate lightweight index —
not the main `service_listings` index.

**Tasks:**
- [ ] Create `search_suggestions_v1` Elasticsearch index — separate from `service_listings`
  - Documents: category names, subcategory names, popular skill names, popular tag names, top search query terms from `search_queries`
- [ ] `SuggestionIndexBuilder` — builds suggestion documents
- [ ] `GET /api/v1/search/suggestions?q=` — returns up to 10 suggestions (min 2 chars)
- [ ] Cached: Redis, 5-minute TTL per query string
- [ ] `php artisan search:rebuild-suggestions` — rebuilds suggestion index from taxonomy + search_queries
- [ ] Run `search:rebuild-suggestions` as a weekly scheduled job
- [ ] Pest: query returns suggestions, min 2 chars enforced, results are cached on second request

**Acceptance criteria:**
- `GET /api/v1/search/suggestions?q=la` returns suggestions including "Laravel"
- A query of 1 character returns HTTP 422
- A second identical request within 5 minutes is served from Redis

---

---

# Milestone 6 — Admin Panel

**Gate condition:** M3 complete.

---

### M6-01 — Filament core setup and user management

**Branch:** `feature/m6-01-filament-setup`
**Labels:** `admin`, `backend`, `backlog`
**Depends on:** M1-05

**Objective:**
Filament v5 admin panel with authentication, and the user management resource.

**Tasks:**
- [ ] Install Filament v5: `composer require filament/filament`
- [ ] Configure admin panel at `/admin` — separate guard, admin role only
- [ ] Verify non-admin users receive HTTP 403 at `/admin`
- [ ] `UserResource` — list all users, view detail, filter by role/status
  - Action: Suspend (with reason — writes `admin_actions` row, fires `UserSuspendedNotificationEvent`)
  - Action: Ban (with confirmation — writes `admin_actions` row)
  - Action: Restore (sets status back to active)
  - Action: Impersonate (admin views platform as the user — dev and staging only)
- [ ] `AdminAction` model — written on every user action
- [ ] All destructive actions require a confirmation modal before executing
- [ ] Pest: non-admin user gets 403 at /admin, suspend action writes admin_actions row

**Acceptance criteria:**
- A user with `role = buyer` cannot access any `/admin` route
- Suspending a user from Filament creates a row in `admin_actions` with the admin's ID and reason
- The suspension takes effect immediately — the user's next API request returns HTTP 403

---

### M6-02 — Service moderation

**Branch:** `feature/m6-02-service-moderation`
**Labels:** `admin`, `backend`, `backlog`
**Depends on:** M6-01, M1-10

**Objective:**
Admin review queue for service listings submitted for publication.

**Tasks:**
- [ ] `ServiceResource` in Filament — moderation queue view (filter by `status = pending_review`)
  - Action: Approve (transitions to `published`, writes `admin_actions`, dispatches `IndexServiceInSearchJob`)
  - Action: Reject (with reason — transitions to `rejected`, notifies seller)
  - Action: Set `score_boost` (float field — for featuring services in search)
  - View: full service detail including packages, media, FAQs, requirements
- [ ] Services with status `draft` are not shown in the moderation queue
- [ ] Pest: approve action transitions status to published, reject notifies seller, score_boost update reflected in next search index job

**Acceptance criteria:**
- Approving a service from Filament transitions it to `published` and dispatches `IndexServiceInSearchJob`
- Rejecting a service sends an email to the seller with the rejection reason
- Setting `score_boost` to 2.0 causes the service to rank higher in search results

---

### M6-03 — Order and dispute management

**Branch:** `feature/m6-03-order-management`
**Labels:** `admin`, `backend`, `backlog`
**Depends on:** M6-01, M2-04

**Objective:**
Admin visibility into all orders and tools to arbitrate disputes.

**Tasks:**
- [ ] `OrderResource` in Filament — view all orders, filter by status
  - View: full order detail including deliveries, revisions, resolution history
  - Action: Force-cancel (with reason — counts against seller cancellation rate, writes `admin_actions`)
  - Action: Assign admin to escalated resolution (sets `order_resolutions.admin_id`)
  - Action: Resolve dispute (sets outcome, triggers appropriate refund or release)
- [ ] Escalated resolutions appear in a dedicated "Disputes" view (filter: `resolution_status = escalated`)
- [ ] `DisputeWidget` on Filament dashboard — count of open escalated disputes
- [ ] Pest: force-cancel increments seller cancellation rate, assigns correct admin_id on dispute assignment

**Acceptance criteria:**
- Admin force-cancelling an order writes to `admin_actions` and increments `seller_profiles.cancellation_count`
- Escalated disputes appear in the dedicated disputes view within 60 seconds of escalation
- Resolving a dispute triggers the correct financial outcome (refund or escrow release)

---

### M6-04 — Seller applications and financial operations

**Branch:** `feature/m6-04-admin-financial`
**Labels:** `admin`, `backend`, `payment-critical`, `backlog`
**Depends on:** M6-01, M1-07, M3-03

**Objective:**
Admin tools for seller application review, payout management, and platform
financial reporting.

**Tasks:**
- [ ] `SellerApplicationResource` — view submitted applications in `under_review` status
  - View: full application data including fraud check result and profile completeness score
  - Action: Approve (calls `SellerOnboardingApprovalPipeline::approve()`)
  - Action: Reject (with reason textarea — calls `SellerOnboardingApprovalPipeline::reject()`)
  - Filter: flagged applications (fraud_check_result isFlagged = true)
- [ ] `PayoutResource` — view all withdrawals
  - Filter: by status (pending, processing, completed, failed)
  - Action: Retry payout (for failed withdrawals — calls Stripe Connect transfer again)
  - View: full ledger history for any user
- [ ] `RevenueOverviewWidget` on dashboard — total GMV, platform revenue, pending clearance (current month)
- [ ] `PlatformSettings` Filament page — edit fee rates, clearing days, feature flags
  - Changes saved to `feature_flags` table and to `config/payment.php` cache
- [ ] All financial admin actions write to `admin_actions` audit log
- [ ] Pest: approve application calls UpgradeToSellerAction, reject sends email, payout retry calls Stripe

**Acceptance criteria:**
- Approving a seller application from Filament has identical outcome to automated approval
- All fee rate changes in Platform Settings take effect on the next order placed (no deployment needed)
- Payout retry for a failed withdrawal creates a new Stripe Connect transfer

---

### M6-05 — Search analytics and webhook monitoring

**Branch:** `feature/m6-05-admin-monitoring`
**Labels:** `admin`, `backend`, `devops`, `backlog`
**Depends on:** M6-01, M5-04, M3-07

**Objective:**
Admin visibility into search performance, webhook health, and feature flag management.

**Tasks:**
- [ ] `FeatureFlagResource` — list all feature flags, toggle enabled/disabled, set rollout percentage
- [ ] `SearchQueryWidget` — top 20 search queries (last 7 days), zero-result queries count
- [ ] `WebhookHealthWidget` — dead-letter queue count, last successful webhook timestamp
- [ ] Admin action: `Retry all failed webhooks` button — calls `php artisan webhooks:retry-failed`
- [ ] Admin action: `Rebuild search index` button — dispatches `ReindexAllServicesJob` (batch queue)
- [ ] Pest: feature flag toggle changes DB record, search widget returns correct query counts

**Acceptance criteria:**
- Toggling a feature flag in Filament takes effect on the next API call that checks that flag
- The webhook health widget shows a count > 0 when failed events exist in the dead-letter queue
- "Rebuild search index" button dispatches the reindex job and shows a success notification

---

---

# Milestone 7 — Stabilization

**Gate condition:** M6 complete.

---

### M7-01 — Performance profiling and N+1 elimination

**Branch:** `chore/m7-01-performance`
**Labels:** `backend`, `backlog`
**Depends on:** M6 complete

**Objective:**
Profile all high-traffic endpoints and eliminate N+1 queries.
Establish baseline response time benchmarks.

**Tasks:**
- [ ] Enable `Model::preventLazyLoading()` in `AppServiceProvider` for non-production environments
- [ ] Run all feature tests — fix any lazy loading violations
- [ ] Profile with Telescope: identify all endpoints with > 5 queries
- [ ] Fix N+1s with eager loading — document each fix in the PR
- [ ] Establish response time benchmarks for the 10 highest-traffic endpoints:
  - `GET /api/v1/search/services`
  - `GET /api/v1/me/orders/selling`
  - `GET /api/v1/conversations`
  - `GET /api/v1/me/analytics/overview`
  - (and 6 others identified during profiling)
- [ ] All benchmarked endpoints respond in under 200ms on seeded dataset of 10,000 orders
- [ ] Document benchmarks in `docs/process/performance-benchmarks.md`

**Acceptance criteria:**
- `Model::preventLazyLoading()` raises zero exceptions during the full test suite
- All 10 benchmarked endpoints respond under 200ms
- No endpoint makes more than 10 database queries (verified with query log)

---

### M7-02 — Security audit

**Branch:** `chore/m7-02-security-audit`
**Labels:** `backend`, `security`, `backlog`
**Depends on:** M6 complete

**Objective:**
Verify all critical security controls are in place before production.

**Tasks:**
- [ ] Verify every controller method calls `$this->authorize()` (automated grep check)
- [ ] Verify no raw SQL queries exist without parameter binding (`DB::select()` without bindings)
- [ ] Verify no sensitive data (passwords, tokens, API keys, PII) is logged anywhere
- [ ] Verify all financial endpoints have rate limiting middleware
- [ ] Verify Stripe webhook signature is validated on every webhook request
- [ ] Verify the `ledger_entries` table has no `updated_at` column (immutability check)
- [ ] Verify the `admin_actions` table has no `updated_at` column (immutability check)
- [ ] Run `roave/security-advisories` check: `composer audit`
- [ ] Review all `@architecture-exception` comments — confirm all are resolved
- [ ] Document audit results in `docs/process/security-audit-m7.md`

**Acceptance criteria:**
- `grep -r "authorize" app/Http/Controllers/ | wc -l` equals the total number of public controller methods
- `composer audit` reports no known vulnerabilities
- All `@architecture-exception` comments have been resolved or do not exist

---

### M7-03 — Load testing

**Branch:** `chore/m7-03-load-testing`
**Labels:** `devops`, `backlog`
**Depends on:** M7-01

**Objective:**
Verify the system handles expected peak traffic without degradation.

**Tasks:**
- [ ] Install k6 or Artillery for load testing
- [ ] Write load test scripts for the 3 critical flows:
  - Search: 100 concurrent users, 1000 requests/min to `GET /api/v1/search/services`
  - Order placement: 20 concurrent users placing orders simultaneously
  - Messaging: 50 concurrent users sending messages
- [ ] Run load tests against staging environment with production-sized dataset (10k orders, 1k sellers, 5k services)
- [ ] Document results: p50, p95, p99 response times; error rate; Elasticsearch query times
- [ ] Identify and resolve any bottlenecks discovered
- [ ] Document final results in `docs/process/load-test-results-m7.md`

**Acceptance criteria:**
- Search: p95 under 300ms, zero errors at 100 concurrent users
- Order placement: p95 under 500ms, zero duplicate charges
- Error rate across all tests: under 0.1%

---

### M7-04 — Documentation review and final freeze

**Branch:** `chore/m7-04-docs-review`
**Labels:** `documentation`, `backlog`
**Depends on:** M6 complete

**Objective:**
Review and update all architecture documents. Confirm they reflect
the implemented system, not the designed system.

**Tasks:**
- [ ] Review `PROJECT_RULES_AND_EXECUTION.md` — verify all 20 sections still accurate
- [ ] Update any section that does not match the implemented codebase
- [ ] Review `where-does-this-go.md` — add any patterns that emerged during implementation
- [ ] Review `onboarding-guide.md` — update with any lessons learned during Milestones 1–6
- [ ] Review `elasticsearch-ops.md` — verify Artisan commands match what was actually built
- [ ] Run the quarterly architecture review (see Rule 8 in `rules.md`)
- [ ] Confirm Architecture Steward has reviewed and signed off
- [ ] Update Document Control version in `PROJECT_RULES_AND_EXECUTION.md` to reflect any changes

**Acceptance criteria:**
- CI docs-freshness check passes (all architecture docs committed within 90 days)
- Architecture Steward has committed a sign-off comment to `rules.md`
- No rule in the engineering contract references a pattern that was not actually implemented

---

### M7-05 — Production infrastructure preparation

**Branch:** `chore/m7-05-infra-prep`
**Labels:** `devops`, `backlog`
**Depends on:** M7-02

**Objective:**
Prepare the production environment configuration. Does not include go-live.

**Tasks:**
- [ ] Configure production `.env` (no defaults, all values explicit)
- [ ] Configure MySQL read/write split in `config/database.php`
- [ ] Configure Redis Cluster or Redis Sentinel for production
- [ ] Configure Elasticsearch with production hardware sizing
- [ ] Configure CloudFront CDN in front of S3
- [ ] Configure Sentry DSN for production environment
- [ ] Configure SES production sending limits and domain verification
- [ ] Configure Horizon Supervisor for production worker management
- [ ] Write `docs/process/production-runbook.md`:
  - How to deploy a new release
  - How to run a database migration safely in production
  - How to perform an Elasticsearch index migration
  - How to roll back a failed deployment
  - How to restart Horizon workers
  - Emergency contact list

**Acceptance criteria:**
- `docs/process/production-runbook.md` exists and covers all 6 procedures
- Staging environment uses the production configuration (except DNS and secrets)
- All production environment variables are documented in `.env.example`

---

---

# Milestone 8 — Production Launch

**Gate condition:** M7 complete. Security audit signed off.
Final go/no-go decision by Technical Approver and Architecture Steward.

---

### M8-01 — Pre-launch checklist

**Branch:** `chore/m8-01-pre-launch`
**Labels:** `devops`, `high-priority`, `backlog`
**Depends on:** M7-05

**Objective:**
Execute all pre-launch verification steps. This issue is the formal
go/no-go gate for production deployment.

**Tasks:**
- [ ] All CI jobs pass on the release branch
- [ ] All Milestone 7 issues are closed
- [ ] Security audit document is signed off (`docs/process/security-audit-m7.md`)
- [ ] Load test results are within acceptable thresholds (`docs/process/load-test-results-m7.md`)
- [ ] Production environment verified: all services healthy, all secrets set
- [ ] Stripe production mode enabled and webhook endpoint registered
- [ ] DNS and SSL certificates configured
- [ ] Monitoring alerts configured in Sentry (critical errors, payment failures)
- [ ] `php artisan ledger:reconcile` runs clean on production data migration
- [ ] Rollback procedure documented and tested on staging
- [ ] Technical Approver and Architecture Steward both confirm: "Go for launch"

**Acceptance criteria:**
- All checklist items above are checked off in this issue
- Both sign-offs are added as comments on this issue by the named role-holders
- Zero open `architecture-exception` issues exist

---

### M8-02 — Production deployment

**Branch:** `chore/m8-02-deployment`
**Labels:** `devops`, `high-priority`, `backlog`
**Depends on:** M8-01

**Objective:**
Execute the production deployment following the runbook.

**Tasks:**
- [ ] Enable maintenance mode: `php artisan down`
- [ ] Deploy release to production servers
- [ ] Run migrations: `php artisan migrate --force`
- [ ] Run taxonomy seeders if first deployment: `php artisan db:seed --class=TaxonomySeeder`
- [ ] Run `php artisan search:create-index v1` and `php artisan search:swap-alias v1`
- [ ] Run `php artisan search:reindex-all` (may take several minutes)
- [ ] Start Horizon workers: `php artisan horizon`
- [ ] Start Reverb server: `php artisan reverb:start`
- [ ] Disable maintenance mode: `php artisan up`
- [ ] Verify: health check endpoint returns 200
- [ ] Verify: Sentry receives a test event
- [ ] Verify: Horizon dashboard shows all queues running
- [ ] Verify: place a test order in production (Stripe test mode first, then live)

**Acceptance criteria:**
- Health check endpoint returns HTTP 200 within 60 seconds of `php artisan up`
- A test order completes end-to-end in production: placed → requirements → delivered → completed
- No critical errors appear in Sentry within 30 minutes of deployment

---

### M8-03 — Post-launch monitoring (72-hour watch)

**Branch:** `chore/m8-03-post-launch`
**Labels:** `devops`, `high-priority`, `backlog`
**Depends on:** M8-02

**Objective:**
Active monitoring for 72 hours after go-live. Document and resolve any issues discovered.

**Tasks:**
- [ ] Monitor Sentry error rate every hour for the first 24 hours
- [ ] Monitor Horizon queue depth every hour — no queue should back up beyond 100 items
- [ ] Monitor Elasticsearch dead-letter queue — should remain at zero
- [ ] Monitor `php artisan ledger:reconcile` output daily — should report no discrepancies
- [ ] Log all issues discovered as GitHub issues tagged `bug` and `high-priority`
- [ ] Resolve all critical or high severity bugs within 24 hours
- [ ] After 72 hours: write `docs/process/launch-retrospective.md`:
  - What went smoothly
  - What needed emergency fixes
  - What to improve in the next release cycle

**Acceptance criteria:**
- Zero payment errors or data inconsistencies in the first 72 hours
- `php artisan ledger:reconcile` reports no discrepancies for 3 consecutive days
- `launch-retrospective.md` committed within 5 days of go-live

---

---

## Issue summary by milestone

| Milestone | Issues |
|---|---|
| M0 — Infrastructure & Architecture | M0-01 through M0-08 (8 issues) |
| M1 — User + Taxonomy + Service | M1-01 through M1-10 (10 issues) |
| M2 — Orders + Messaging + Reviews | M2-01 through M2-09 (9 issues) |
| M3 — Payments + Billing | M3-01 through M3-07 (7 issues) |
| M4 — Notifications + Analytics | M4-01 through M4-05 (5 issues) |
| M5 — Search | M5-01 through M5-05 (5 issues) |
| M6 — Admin Panel | M6-01 through M6-05 (5 issues) |
| M7 — Stabilization | M7-01 through M7-05 (5 issues) |
| M8 — Production Launch | M8-01 through M8-03 (3 issues) |
| **Total** | **57 issues** |

---

## Branch naming reference

```
M0:  chore/m0-01-repo-init
     chore/m0-02-code-quality
     chore/m0-03-ci-pipeline
     chore/m0-04-phpstan-rules
     chore/m0-05-architecture-docs
     chore/m0-06-github-setup
     feature/m0-07-shared-kernel
     feature/m0-08-reference-implementations

M1:  db/m1-01-all-migrations
     db/m1-02-taxonomy-seeders
     feature/m1-03-authentication
     feature/m1-04-oauth
     feature/m1-05-roles-policies
     feature/m1-06-seller-onboarding
     feature/m1-07-onboarding-approval
     feature/m1-08-user-profile
     feature/m1-09-taxonomy-api
     feature/m1-10-service-listings

M2:  feature/m2-01-buyer-requests
     feature/m2-02-order-placement
     feature/m2-03-order-lifecycle
     feature/m2-04-resolution-centre
     feature/m2-05-messaging
     feature/m2-06-reviews
     feature/m2-07-feature-flags
     feature/m2-08-seller-levels
     chore/m2-09-observability

M3:  feature/m3-01-stripe-integration
     feature/m3-02-escrow-clearing
     feature/m3-03-seller-payouts
     feature/m3-04-billing
     feature/m3-05-platform-credits
     chore/m3-06-ledger-audit
     chore/m3-07-webhook-reliability

M4:  feature/m4-01-notifications
     feature/m4-02-seller-analytics
     chore/m4-03-queue-workers
     feature/m4-04-email-templates
     chore/m4-05-scheduled-jobs

M5:  feature/m5-01-elasticsearch-setup
     feature/m5-02-search-sync
     feature/m5-03-search-api
     feature/m5-04-search-logging
     feature/m5-05-autocomplete

M6:  feature/m6-01-filament-setup
     feature/m6-02-service-moderation
     feature/m6-03-order-management
     feature/m6-04-admin-financial
     feature/m6-05-admin-monitoring

M7:  chore/m7-01-performance
     chore/m7-02-security-audit
     chore/m7-03-load-testing
     chore/m7-04-docs-review
     chore/m7-05-infra-prep

M8:  chore/m8-01-pre-launch
     chore/m8-02-deployment
     chore/m8-03-post-launch
```

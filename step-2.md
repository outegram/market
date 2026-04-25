# PART 2 — FIVERR DEEP ANALYSIS (DOMAINS 5–8)

  

# DOMAIN 5 — Gig / Services System

  

# Core Principle

The Gig system is not a “listing system.”

It is:

# Productized Service Commerce Engine

A gig is a structured commercial contract template.

It defines:

-   scope
-   delivery expectations
-   pricing boundaries
-   revision policy
-   upsell structure
-   buyer expectations
-   seller accountability
-   dispute evidence baseline

Weak gig architecture creates permanent dispute volume.

This is one of the most critical domains.

  

# Core Gig Model

A gig must be treated as:

Commercial Product  
—not—  
Simple Post

This means:

Never:

title  
description  
price

only.

That architecture fails immediately.

A production-grade gig must support:

-   structured packages
-   scoped deliverables
-   service extras
-   requirements
-   revision boundaries
-   media proof
-   moderation
-   ranking signals
-   lifecycle states
-   abuse controls

  

# Gig Creation Flow

## Step-Based Workflow

Recommended flow:

Step 1 → Overview  
Step 2 → Scope & Pricing  
Step 3 → Description & FAQ  
Step 4 → Requirements  
Step 5 → Gallery & Attachments  
Step 6 → Publish Review

Reason:

reduces incomplete gigs  
improves moderation quality  
improves conversion

Must support:

Draft  
Auto-save  
Versioning  
Recovery

Never force full completion in one form.

  

# Categories & Taxonomy

Correct structure:

Category  
→ Subcategory  
→ Service Type  
→ Skill Tags  
→ Delivery Model

Example:

Design  
→ Logo Design  
→ Minimalist Logo  
→ Brand Identity  
→ Fast Delivery

Do NOT use flat categories.

Why:

-   ranking precision
-   better filtering
-   moderation
-   better recommendations
-   analytics quality

Taxonomy must be admin-controlled.

Never seller-controlled.

  

# Packages System

## Basic / Standard / Premium

This is mandatory.

Each package must define:

-   title
-   scope summary
-   delivery time
-   revision count
-   price
-   included features
-   delivery units
-   commercial usage rights
-   source files included
-   support scope

Example:

Basic → 1 concept  
Standard → 2 concepts  
Premium → full branding kit

This increases AOV significantly.

  

# Pricing Logic

Must support:

## Base Price

Package core pricing

## Dynamic Extras

Examples:

-   fast delivery
-   additional revision
-   source file
-   extra concept
-   commercial rights
-   consultation call

## Quantity-Based Pricing

Examples:

-   1000 words
-   5 pages
-   10 products

## Custom Offers

Seller-generated private offers

## Milestone-Based Orders

For large projects

Pricing must be rule-based.

Never free manual price mutation inside orders.

  

# FAQ System

Purpose:

reduce repetitive pre-sale messaging

Each gig supports:

-   question
-   answer
-   ordering
-   moderation

FAQ also helps:

-   SEO
-   trust
-   conversion

  

# Gallery System

Supports:

-   images
-   videos
-   PDFs
-   preview assets

Rules required:

-   file validation
-   malware scanning
-   size limits
-   moderation queue
-   copyright checks
-   abuse reporting

Media is a trust surface.

  

# Requirements System

Critical for disputes.

Buyer must submit:

## Structured Requirements

Examples:

-   brand name
-   preferred colors
-   reference examples
-   file formats needed

Supports:

-   text
-   select
-   checkbox
-   file upload
-   conditional questions

Without structured requirements:

delivery disputes explode.

  

# Delivery Time Logic

Must be explicit.

Never inferred.

Supports:

-   package delivery time
-   extra fast delivery
-   business day logic
-   holiday awareness
-   pause during buyer pending states

Time calculation must be deterministic.

Disputes depend on this.

  

# Revision Rules

Must be finite.

Never:

Unlimited revisions

without strong restrictions.

Need:

-   included revision count
-   paid extra revisions
-   revision scope boundaries
-   revision abuse prevention

Unlimited revisions destroy seller retention.

  

# Gig SEO + Ranking Metadata

Fields:

-   search title
-   slug
-   metadata
-   tags
-   intent keywords
-   semantic category mapping
-   trust score relevance

Gig ranking must use:

-   quality signals
-   seller trust
-   conversion history

—not keyword stuffing.

  

# Gig Moderation Workflow

States:

Draft  
Pending Review  
Approved  
Published  
Paused  
Restricted  
Rejected  
Archived  
Soft Deleted

Never:

published = true

This is incorrect architecture.

Moderation includes:

-   automated checks
-   manual review
-   copyright review
-   policy review
-   fraud review

  

# Abuse Prevention

Must detect:

-   duplicate gigs
-   spam gigs
-   copied descriptions
-   stolen portfolios
-   prohibited services
-   policy violations
-   manipulated keywords

This requires moderation + trust signals.

  

# Required Database Concepts

Tables:

-   gigs
-   gig\_versions
-   gig\_packages
-   gig\_package\_features
-   gig\_extras
-   gig\_requirements
-   gig\_requirement\_answers
-   gig\_faqs
-   gig\_media
-   gig\_tags
-   gig\_categories
-   gig\_moderation\_reviews
-   gig\_status\_history
-   custom\_offers

  

# DOMAIN 6 — Search & Discovery System

  

# Core Principle

Search is not filtering.

Search is:

# Revenue Engine

Search ranking directly controls:

-   GMV
-   seller growth
-   trust
-   buyer retention
-   fraud visibility
-   platform quality

Bad search destroys the marketplace.

  

# Search Engine Behavior

Ranking should not be:

latest gigs first

or

highest price first

This is amateur architecture.

Ranking must be multi-factor.

  

# Ranking Signals

Core ranking inputs:

## Seller Trust

-   level
-   dispute rate
-   completion rate
-   warning history
-   fraud score

## Gig Performance

-   impressions
-   CTR
-   conversion rate
-   repeat buyers
-   review quality

## Operational Reliability

-   response rate
-   on-time delivery
-   cancellation rate
-   revision abuse

## Relevance

-   category match
-   keyword intent
-   buyer personalization

## Buyer Context

-   location
-   language
-   budget
-   previous behavior

Search is behavioral ranking.

Not SQL LIKE queries.

  

# Filters System

Must support:

-   category
-   subcategory
-   seller level
-   delivery time
-   budget
-   language
-   country
-   online status
-   pro seller
-   top rated
-   local seller
-   subscription availability
-   service options

Filters must be index-driven.

Never full-scan queries.

  

# Tags System

Use controlled + flexible tags.

Not free-text chaos.

Need:

-   admin tags
-   seller tags
-   semantic tags
-   moderation checks

Tags improve:

-   recommendations
-   ranking quality
-   internal analytics

  

# Personalized Recommendations

Must support:

## Behavioral Signals

-   viewed gigs
-   abandoned checkout
-   completed orders
-   saved gigs
-   repeat categories

## Similar Buyer Modeling

## Seller Affinity

## Seasonal Intent

## High-Intent Recovery

Examples:

Continue where you left off  
Recommended for your project  
Similar to your recent order

This improves retention.

  

# Trending Services

Must be controlled.

Not only:

most viewed

Need:

-   conversion quality
-   fraud filtering
-   review quality
-   temporary spikes detection

Prevent manipulation.

  

# Featured Gigs

Manual + algorithmic.

Can include:

-   platform campaigns
-   seasonal promotions
-   verified sellers
-   strategic category boosting

Must be transparent internally.

Not arbitrary.

  

# Sponsored Placements

Future monetization layer.

Must be:

## Clearly Separated

Never hidden ranking manipulation.

Need:

-   ad disclosure
-   fraud prevention
-   ROI analytics
-   budget controls
-   abuse detection

Trust must not be sold invisibly.

  

# Search Infrastructure

Recommended:

Initial stage:

MySQL + optimized indexes + cached ranking snapshots

Later scale:

Dedicated search engine

But not before justified.

Do not prematurely over-engineer.

  

# Required Database Concepts

Tables:

-   search\_impressions
-   search\_clicks
-   gig\_rank\_snapshots
-   buyer\_preferences
-   saved\_gigs
-   recommendation\_logs
-   sponsored\_placements
-   featured\_campaigns
-   ranking\_audit\_logs

Search must be auditable.

  

# DOMAIN 7 — Orders System

  

# Core Principle

Orders are not “purchases.”

Orders are:

# Controlled Delivery Contracts

This domain is the operational heart of the marketplace.

Orders control:

-   revenue
-   disputes
-   trust
-   legal responsibility
-   refunds
-   retention

This must be state-machine driven.

Never loosely coded.

  

# Order Placement Flow

Flow:

Select Gig  
→ Select Package  
→ Add Extras  
→ Checkout  
→ Payment Authorization  
→ Requirement Submission  
→ Seller Starts Work  
→ Delivery  
→ Review / Revision / Completion

No shortcuts.

Each stage must be tracked.

  

# Checkout Integrity

At order creation:

freeze:

-   pricing
-   package definition
-   gig version
-   revision limits
-   delivery expectations
-   tax calculation
-   fees
-   platform commission

Never reference mutable gig data after purchase.

Orders must be immutable contracts.

  

# Milestones

Required for:

-   large projects
-   high-ticket services
-   enterprise work

Each milestone must have:

-   scope
-   delivery date
-   acceptance state
-   release conditions
-   dispute boundaries

Milestones reduce refund risk.

  

# Requirement Submission

Order must not start until requirements are submitted.

State:

Awaiting Buyer Requirements

Timer logic must respect this.

Seller should not be penalized before buyer submits.

Critical rule.

  

# Delivery Flow

Seller submits:

-   delivery files
-   notes
-   version metadata
-   structured delivery proof

Supports:

-   partial delivery
-   milestone delivery
-   revision response

Delivery must be auditable.

  

# Revisions System

State transitions:

Delivered  
→ Revision Requested  
→ Revision In Progress  
→ Redelivered  
→ Accepted

Need:

-   revision limits
-   abuse controls
-   evidence trail

  

# Acceptance & Auto Completion

Buyer can:

## Accept manually

or system:

## Auto-completes after review window

This prevents buyer ghosting.

Example:

3-day auto-complete window

Must be configurable.

  

# Late Delivery Handling

Need:

-   grace logic
-   late flags
-   seller performance impact
-   dispute triggers
-   auto-extension rules where justified

Late delivery affects seller level system.

Must integrate.

  

# Cancellation Rules

Never allow unrestricted cancellation.

Need rules for:

-   before work starts
-   after work starts
-   after delivery
-   after dispute
-   abuse patterns

Cancellation must be policy-driven.

Not support-agent improvisation.

  

# Refund Logic

Supports:

-   full refund
-   partial refund
-   milestone refund
-   dispute refund
-   wallet refund
-   original payment refund

Financial consistency is mandatory.

  

# Order Status Lifecycle

Recommended:

Pending Payment  
Awaiting Requirements  
In Progress  
Delivered  
Revision Requested  
Under Review  
Completed  
Cancelled  
Disputed  
Refunded  
Partially Refunded  
Late  
On Hold

Never:

status = active

Too vague.

  

# Required Database Concepts

Tables:

-   orders
-   order\_items
-   order\_snapshots
-   order\_requirements
-   order\_deliveries
-   order\_revisions
-   order\_milestones
-   order\_status\_history
-   cancellations
-   refunds
-   dispute\_flags

State history is mandatory.

  

# DOMAIN 8 — Messaging System

  

# Core Principle

Messaging is not chat.

Messaging is:

# Transactional Communication Infrastructure

It affects:

-   conversion
-   dispute evidence
-   fraud detection
-   support intervention
-   abuse control

This system must be governed.

Not “free chat.”

  

# Messaging Types

Must separate:

## Pre-order Inbox

buyer ↔ seller before purchase

## Order Chat

communication tied to specific order

## System Messages

platform-generated events

## Admin Intervention Messages

support + dispute resolution communication

Do not merge all blindly.

Context matters.

  

# Inbox System

Features:

-   conversation threads
-   attachment support
-   quote requests
-   custom offer negotiation
-   spam controls
-   moderation visibility

Inbox should feed conversion.

Not just chat.

  

# Direct Messaging Rules

Need restrictions:

-   contact info sharing detection
-   external payment attempts detection
-   suspicious links detection
-   spam detection
-   scam behavior monitoring

Platforms lose revenue through off-platform leakage.

This must be aggressively monitored.

  

# Order Chat

This is legal evidence.

Must support:

-   delivery references
-   revision requests
-   requirement clarifications
-   milestone discussions
-   dispute evidence export

Order chat must be immutable.

No silent deletions.

  

# Attachments

Supports:

-   documents
-   images
-   archives (restricted)
-   previews

Must include:

-   malware scanning
-   MIME validation
-   size limits
-   retention policy
-   legal retention controls

Never trust uploaded files.

  

# Read Status

Needed for:

-   dispute evidence
-   delivery confirmation
-   support mediation

Track:

-   sent
-   delivered
-   seen

This is operational evidence.

  

# Spam Prevention

Need:

-   rate limiting
-   suspicious behavior scoring
-   repeated message detection
-   template abuse detection
-   cold outreach limits

Spam destroys trust fast.

  

# Abuse Prevention

Detect:

-   harassment
-   threats
-   hate speech
-   fraud attempts
-   manipulation
-   extortion
-   policy violations

Need:

-   report flow
-   moderation review
-   escalation rules

  

# Moderation

Must support:

-   hidden review tools
-   evidence locking
-   admin intervention
-   escalation workflows
-   moderation notes

Users should not see internal moderation actions.

  

# Message Retention

Never immediate deletion.

Need:

-   legal retention rules
-   dispute retention rules
-   audit retention
-   GDPR-aware deletion strategy

Compliance matters.

  

# Required Database Concepts

Tables:

-   conversations
-   conversation\_participants
-   messages
-   message\_attachments
-   message\_reads
-   message\_flags
-   spam\_scores
-   moderation\_cases
-   communication\_audit\_logs

Messaging must be reviewable.

  

# PART 2 COMPLETE

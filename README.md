# Competitive Intelligence Platform

[![TypeScript](https://img.shields.io/badge/TypeScript-5.4-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Next.js](https://img.shields.io/badge/Next.js-14-000000?logo=next.js&logoColor=white)](https://nextjs.org/)
[![BullMQ](https://img.shields.io/badge/BullMQ-5-E74C3C?logo=redis&logoColor=white)](https://docs.bullmq.io/)
[![Supabase](https://img.shields.io/badge/Supabase-PostgreSQL-3ECF8E?logo=supabase&logoColor=white)](https://supabase.com/)
[![Playwright](https://img.shields.io/badge/Playwright-1.48-2EAD33?logo=playwright&logoColor=white)](https://playwright.dev/)

A production data engineering system that automates competitive pricing analysis across 31 retail chains in 14 states. Built to replace a manual process that took my team 2+ weeks per cycle, now runs on autopilot every 6 hours with real-time Telegram alerting.

> **Note:** This is a showcase repo. The production codebase is private due to proprietary business data. This README documents the architecture, engineering decisions, and technical challenges I solved while building it.

---

## The Problem

The pricing team was manually visiting 25+ competitor websites, copying product names and prices into spreadsheets, and comparing them side by side. For every state. Every two weeks. Six separate spreadsheets, no normalization, no consistency. By the time the analysis was done, half the data was already stale.

## What I Built

A fully automated pipeline that:
- Reverse-engineers 3 proprietary retail API platforms (REST, GraphQL, and Algolia)
- Scrapes 50,000+ products across 65+ store locations in 14 states
- Normalizes messy, inconsistent data into a single clean schema
- Runs OCR on competitor promotional images to extract discount details
- Classifies promotional threats by severity (CRITICAL, HIGH, MEDIUM, LOW)
- Refreshes everything on automated 6-hour cycles with Telegram status alerts
- Serves it all through a C-suite dashboard with XLSX export

The team went from 6 spreadsheets and 2-week turnarounds to one dashboard with same-day data.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     SCHEDULER (Cron)                         │
│       65 jobs staggered 30s apart on 6-hour cycles           │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                  JOB QUEUE (BullMQ + Redis)                  │
│                                                              │
│  Concurrency: 3   │   Retry: 3x exponential backoff         │
│  Dedup: deterministic job IDs (no duplicate execution)       │
│  Retention: 50 completed, 20 failed (auto-pruned)            │
│                                                              │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐                │
│  │ Job 1  │ │ Job 2  │ │ Job 3  │ │ Job N  │  ... (65)      │
│  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘                │
└──────┼──────────┼──────────┼──────────┼──────────────────────┘
       │          │          │          │
       ▼          ▼          ▼          ▼
┌──────────────────────────────────────────────────────────────┐
│                    SCRAPER ENGINE                             │
│                                                              │
│  Strategy A: Direct API (fastest, 60-90s per chain)          │
│  Strategy B: Browser intercept (6 parallel contexts)         │
│  Strategy C: Full Playwright (fallback, 200-300s)            │
│                                                              │
│  Auto-fallback: A fails → B → C (no manual intervention)    │
│                                                              │
│  ┌──────────────┐ ┌──────────────┐ ┌───────────────────┐    │
│  │  REST Client  │ │GraphQL Client│ │ Algolia Client    │    │
│  │ (Platform A)  │ │ (Platform B) │ │ (Platform C)      │    │
│  └──────┬────────┘ └──────┬───────┘ └────────┬──────────┘    │
│         │                 │                   │              │
│         ▼                 ▼                   ▼              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │            Browserless Chrome Pool (4 sessions)      │    │
│  │     Cloudflare bypass, cookie extraction, stealth    │    │
│  └──────────────────────────┬───────────────────────────┘    │
└─────────────────────────────┼────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                DATA TRANSFORMATION LAYER                     │
│                                                              │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────┐  │
│  │   Category       │  │   Brand Alias    │  │   Price    │  │
│  │   Normalizer     │  │   Engine         │  │ Calculator │  │
│  │                  │  │                  │  │            │  │
│  │  50+ variations  │  │  500+ brands     │  │ Tax norm,  │  │
│  │  → 15 canonical  │  │  resolved to     │  │ member vs  │  │
│  │  with confidence │  │  canonical names │  │ regular,   │  │
│  │  scoring         │  │                  │  │ $/g calc   │  │
│  └─────────────────┘  └──────────────────┘  └────────────┘  │
│                                                              │
│  ┌─────────────────┐  ┌──────────────────┐                   │
│  │   Size Parser   │  │ Product Identity │                   │
│  │                 │  │                  │                   │
│  │ Fractions, g,   │  │ Deterministic    │                   │
│  │ mg, oz, Unicode │  │ slugs for clean  │                   │
│  │ → grams         │  │ upserts          │                   │
│  └─────────────────┘  └──────────────────┘                   │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│               DATABASE (Supabase / PostgreSQL)               │
│                                                              │
│  Products │ Promotions │ Price History │ Scrape Logs │ Stores│
│  21 migrations │ Row Level Security │ Stored procedures      │
└─────────────────────────────┬────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
┌──────────────────┐ ┌───────────────┐ ┌──────────────────┐
│   DASHBOARD      │ │   TELEGRAM    │ │   XLSX EXPORT    │
│   (Next.js 14)   │ │   ALERTS      │ │                  │
│                  │ │               │ │ One-click Excel  │
│ 21 routes        │ │ Cycle summary │ │ for C-suite      │
│ 22 API endpoints │ │ Failure alerts│ │ offline analysis  │
│ 18 users + auth  │ │ Zero-product  │ │                  │
│ State scoping    │ │ warnings      │ │                  │
└──────────────────┘ └───────────────┘ └──────────────────┘
```

---

## Technical Challenges

### 1. Reverse-Engineering Proprietary APIs

Each retail platform uses a different technology stack with no public documentation:

- **Platform A (REST):** Required intercepting authenticated session cookies, handling dynamic CSRF tokens, and navigating a multi-step session initialization flow before any data endpoints would respond.

- **Platform B (GraphQL):** Undocumented schema with nested product fragments. Had to reconstruct queries by analyzing network traffic, then handle pagination through cursor-based responses with inconsistent page sizes.

- **Platform C (Algolia):** Search-as-a-service with application-specific API keys embedded in client-side JavaScript. Required extracting runtime configuration, understanding their faceted search model, and implementing proper index targeting across multiple store locations.

### 2. Cloudflare and Anti-Bot Protection

Several chains run behind Cloudflare with JavaScript challenges. I built a Playwright browser pool that:
- Manages persistent browser contexts with cookie jars
- Handles session rotation to avoid rate limiting
- Passes JavaScript challenges through real browser execution
- Uses stealth plugins to avoid headless detection
- Implements 30-second job staggering to stay under detection thresholds

The tiered strategy system means the fastest method is always tried first. Direct API calls (Strategy A) finish in 60-90 seconds per chain. If an API breaks or auth changes, the system automatically falls back to browser interception (B) or full Playwright automation (C) without anyone having to touch the code.

### 3. Data Normalization at Scale

This was honestly harder than the scraping itself.

**Category problem:** 50+ different ways to say the same thing across platforms. The normalizer handles exact matches, fuzzy matches, and hierarchical category resolution with tiered confidence scoring. Each mapping carries a confidence level so downstream analytics can weight uncertain matches appropriately.

**Brand problem:** 500+ brands with inconsistent naming across 31 chains. The brand alias engine resolves all variations to canonical brand names using exact matching, case-insensitive matching, and configurable alias tables.

**Size problem:** One chain uses `1/8oz`, another uses `3.5g`, edibles come in `100mg`, beverages in `30ml`. The size parser handles fractions, Unicode characters (⅛), English words ("eighth"), and every unit variation, normalizing everything to grams for consistent price-per-gram comparison.

**Price problem:** Some APIs return prices with tax, some without. Some show member pricing, some regular, some both with different labels. The price calculator normalizes everything to a consistent pre-tax, regular-price basis for accurate comparison.

### 4. Promotional Threat Analysis

Competitor promotions are scraped from aggregator sites and individual chain APIs. The tricky part: many promotions are image-only (no structured text). I integrated Google Cloud Vision OCR to extract discount details from promotional images, then run the extracted text through a classifier that determines:

- **Discount type:** Percentage, dollar amount, BOGO, bundle, tiered, free item
- **Threat level:** CRITICAL (50%+ off), HIGH (30%+ or BOGO), MEDIUM (15-30%), LOW (under 15%)
- **Category overlap:** Which of our product categories are directly threatened

OCR results are cached in the database to avoid repeat API calls. If OCR fails, the system falls back to any available text metadata.

### 5. Job Deduplication and Cycle Tracking

65 jobs run per cycle across 3 concurrent workers. BullMQ job IDs are deterministic, so if cron fires while a manual trigger is already running, the duplicate collapses into the existing job. No wasted compute, no race conditions.

An in-memory cycle tracker collects results from all 65 jobs and fires a consolidated Telegram summary when the last one completes (or after a 4-hour timeout). One message per cycle instead of 65 separate notifications.

### 6. Reliability at Scale

Running 65 scrapers on a schedule means something is always breaking. An API changes, a session expires, a rate limit hits, a page layout shifts. The system handles this through:

- **Per-chain error isolation:** One scraper failing doesn't take down the others
- **Automatic retry with exponential backoff:** 60s, 180s, 540s (3 attempts)
- **Job staggering:** 30-second delays between jobs to avoid burst load
- **Dead letter handling:** Jobs that fail after retries get logged with full error context
- **Telegram circuit breaker:** Disables notifications after 3 consecutive send failures, resets after 30 minutes
- **Zero-product alerts:** If a scraper returns 0 products, Telegram fires immediately (the scraper is probably broken)
- **Dashboard monitoring:** Real-time job status, success rates, error logs, manual re-trigger

---

## Queue Architecture

BullMQ manages all scraping workloads with Redis (Upstash for cloud environments, local Redis on the VPS).

| Config | Value |
|--------|-------|
| Concurrency | 3 parallel workers |
| Retry | 3 attempts, exponential backoff (60s → 180s → 540s) |
| Retention | Last 50 completed, last 20 failed |
| Deduplication | Deterministic job IDs per chain+state |
| Staggering | 30s between job submissions |
| Cycle timeout | 4 hours |
| Scheduling | Cron: every 6 hours (products), twice daily (store hours), weekly (brand snapshots) |

---

## Dashboard

Next.js 14 full-stack application serving 18 users across the pricing team.

- **State Scoping:** Cookie-based state selector (14 states). Every query filters server-side.
- **Pricing Analytics:** Median price, price-per-gram, percent on sale, per-chain comparison
- **Promotion Feed:** Ranked by threat level. CRITICAL promotions surface first.
- **XLSX Export:** One-click Excel generation for offline analysis and executive reporting
- **Scraper Status:** Real-time job monitoring with manual trigger capability
- **Auth:** HMAC-signed session tokens, 7-day expiry, httpOnly cookies

---

## Monorepo Structure

```
competitive-intel-platform/
├── packages/
│   ├── worker/                    # BullMQ job processing engine
│   │   ├── src/
│   │   │   ├── scraper-engine/    # API clients per platform
│   │   │   ├── competitor-scrapers/   # Chain-specific configs
│   │   │   ├── notifications/     # Telegram bot + cycle tracker
│   │   │   ├── cron/              # Schedule definitions
│   │   │   └── __tests__/         # ~350 tests
│   │
│   ├── scraper/                   # Promotion scraper + OCR pipeline
│   │   ├── scrapers/              # Promo aggregator modules
│   │   ├── parsers/               # Discount classification
│   │   ├── utils/                 # OCR service, rate limiter
│   │   └── __tests__/             # ~180 tests
│   │
│   └── dashboard/                 # Next.js 14 web application
│       ├── src/app/               # 21 routes + 22 API endpoints
│       ├── src/lib/               # Auth, exports, data transforms
│       └── __tests__/             # ~260 tests
│
├── supabase/migrations/           # 21 SQL migrations
├── docker/                        # Docker Compose (3-service stack)
└── .github/workflows/             # CI/CD
```

---

## Stack

| Layer | Technology |
|-------|-----------|
| Language | TypeScript 5.4 (entire codebase) |
| Runtime | Node.js 18+ |
| Frontend | Next.js 14, React 18, Tailwind CSS |
| Job Queue | BullMQ 5.28 on Redis / Upstash |
| Browser Automation | Playwright 1.48 + stealth plugin |
| Chrome Pool | Browserless (4 concurrent sessions, 4 GB) |
| Database | Supabase PostgreSQL with RLS |
| OCR | Google Cloud Vision |
| Notifications | Telegram Bot API |
| Validation | Zod |
| Testing | Vitest (790 tests, 29 suites) |
| Infrastructure | Docker Compose on Hetzner VPS |
| Dashboard Hosting | Render |

---

## Scale

| Metric | Value |
|--------|-------|
| Products tracked | 50,000+ per cycle |
| Retail chains | 31 |
| Store locations | 65+ |
| States covered | 14 |
| API platforms reversed | 3 (REST, GraphQL, Algolia) |
| Scrape cycle | Every 6 hours + on-demand |
| Concurrent workers | 3 |
| Browser sessions | 4 pooled |
| Brand aliases | 500+ |
| Category variations | 50+ → 15 canonical |
| Dashboard users | 18 |
| Test suite | 790 tests, 29 suites |
| Database migrations | 21 |

---

## Testing

790 tests across 29 suites covering the full pipeline from raw API response to dashboard rendering.

| Package | Suites | Tests | What's Covered |
|---------|--------|-------|----------------|
| Worker | 12 | ~350 | Category normalization, price calculation, product validation, job processing |
| Scraper | 8 | ~180 | Discount parsing, OCR validation, rate limiting, threat classification |
| Dashboard | 9 | ~260 | Size parsing, deal formatting, export pipeline, health calculations |

---

## Deployment

| Service | Platform | Notes |
|---------|----------|-------|
| Dashboard | Render | Next.js, auto-deploy from main |
| Worker | Hetzner VPS | Docker Compose (worker + Redis + Browserless) |
| Database | Supabase | Managed PostgreSQL, daily backups, RLS |
| Queue | Redis (local) / Upstash (cloud) | BullMQ backing store |
| Chrome Pool | Browserless | 4 concurrent sessions, 4 GB memory |

Docker Compose orchestrates three services on the VPS: Browserless for headless Chrome, Redis for job queuing, and the TypeScript worker consuming from BullMQ. The dashboard runs separately on Render with its own database and Redis connections for job triggering.

---

## What I Learned

- **Reverse engineering is detective work.** No documentation means you're reading network traffic, testing assumptions, and sometimes just guessing until the response makes sense. Patience matters more than cleverness.

- **Data normalization is where the real value lives.** Anyone can hit an API endpoint. Making the data from 31 different sources actually comparable and useful, that's the hard part, and that's what the team cares about.

- **Reliability beats speed.** I spent more time on error handling, retry logic, and monitoring than on the actual scraping code. A scraper that works 95% of the time is useless if you can't trust which 5% failed.

- **Build for the people who use it.** The dashboard isn't fancy. It's fast, it's clear, and it shows the team exactly what they need. That matters more than any technical architecture.

---

## Why This Repo is a Showcase

The production codebase contains proprietary business logic, API configurations, and competitive data that belongs to my employer. This repo documents the engineering work without exposing anything sensitive.

If you want to see my code, check out my other projects. The [Hyperliquid Trading Simulator](https://github.com/claygeo/hyperliquid-trading-sim) uses similar patterns (WebSocket data ingestion, real-time processing, state management) and is fully open source.

---

*Built by [Clayton George](https://github.com/claygeo)*

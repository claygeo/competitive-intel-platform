# Competitive Intelligence Platform

A production data engineering system that automates competitive pricing analysis across 31 retail chains in 5 states. Built to replace a manual process that took my team 2+ weeks per cycle — now runs on autopilot every 4 hours.

> **Note:** This is a showcase repo. The production codebase is private due to proprietary business data. This README documents the architecture, engineering decisions, and technical challenges I solved while building it.

---

## The Problem

My pricing team was manually visiting 25+ competitor websites, copying product names and prices into spreadsheets, and comparing them side by side. For every state. Every two weeks. Six separate spreadsheets, no normalization, no consistency. By the time the analysis was done, half the data was already stale.

## What I Built

A fully automated pipeline that:
- Reverse-engineers 3 proprietary retail API platforms (REST, GraphQL, and Algolia)
- Scrapes 50,000+ products across 31 retail chains in 5 states
- Normalizes messy, inconsistent data into a single clean schema
- Refreshes everything on automated 4-hour cycles
- Serves it all through a real-time dashboard my team uses daily

The team went from 6 spreadsheets and 2-week turnarounds to one dashboard with same-day data.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      SCHEDULER (Cron)                       │
│            Triggers scrape jobs on 4-hour cycles            │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    JOB QUEUE (BullMQ + Redis)               │
│                                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │ Chain 1 │  │ Chain 2 │  │ Chain 3 │  │ Chain N │  ...    │
│  │  Job    │  │  Job    │  │  Job    │  │  Job    │         │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘         │
└───────┼────────────┼────────────┼────────────┼──────────────┘
        │            │            │            │
        ▼            ▼            ▼            ▼
┌──────────────────────────────────────────────────────────────┐
│                    SCRAPER ENGINE                            │
│                                                              │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐      │
│  │  REST Client  │ │GraphQL Client│ │ Algolia Client   │     │
│  │  (Platform A) │ │ (Platform B) │ │ (Platform C)     │     │
│  └──────┬───────┘ └──────┬───────┘ └────────┬──────────┘     │
│         │                │                   │               │
│         ▼                ▼                   ▼               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              Playwright Browser Pool                │     │
│  │     (Session handling, Cloudflare bypass, cookies)  │     │
│  └─────────────────────────┬───────────────────────────┘     │
└────────────────────────────┼─────────────────────────────────┘
                             │
                             ▼
┌───────────────────────────────────────────────────────────────┐
│                  DATA TRANSFORMATION LAYER                    │
│                                                               │
│  ┌───────────────────┐   ┌───────────────────────────────┐    │
│  │ Category          │   │ Brand Alias Engine            │    │
│  │ Normalizer        │   │                               │    │
│  │                   │   │ "Select"       ─┐             │    │
│  │ "Edibles"    ─┐   │   │ "SELECT ELITE" ─┤→ Select     │    │
│  │ "Edibles -    │   │   │ "Select Elite" ─┘             │    │
│  │  Gummies" ────┤→  │   │                               │    │
│  │ "Gummy"       │   │   │ Resolves 500+ brands across   │    │
│  │ "Food"  ──────┘   │   │ inconsistent naming schemes   │    │
│  │                   │   │                               │    │
│  │ Maps 50+ naming   │   └───────────────────────────────┘    │
│  │ variations into   │                                        │
│  │ unified categories│  ┌───────────────────────────────┐     │
│  └───────────────────┘  │ Price Calculator              │     │
│                         │ Tax normalization, member vs  │     │
│                         │ regular pricing, unit pricing │     │
│                         └───────────────────────────────┘     │
└────────────────────────────┬──────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                   DATABASE (Supabase / PostgreSQL)          │
│                                                             │
│  Products │ Brands │ Categories │ Price History │ Chains    │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    DASHBOARD (Next.js)                      │
│                                                             │
│  Real-time competitive pricing views │ Queue monitoring     │
│  Cross-chain comparison tools        │ Job status tracking  │
│  Category & brand filtering          │ Error reporting      │
└─────────────────────────────────────────────────────────────┘
```

---

## Technical Challenges

### 1. Reverse-Engineering Proprietary APIs

Each retail platform uses a different technology stack with no public documentation:

- **Platform A (REST):** Required intercepting authenticated session cookies, handling dynamic CSRF tokens, and navigating a multi-step session initialization flow before any data endpoints would respond.

- **Platform B (GraphQL):** Undocumented schema with nested product fragments. Had to reconstruct queries by analyzing network traffic, then handle pagination through cursor-based responses with inconsistent page sizes.

- **Platform C (Algolia):** Search-as-a-service with application-specific API keys embedded in client-side JavaScript. Required extracting runtime configuration, understanding their faceted search model, and implementing proper index targeting across multiple store locations.

### 2. Cloudflare & Anti-Bot Protection

Several chains run behind Cloudflare with JavaScript challenges. I built a Playwright browser pool that:
- Manages persistent browser contexts with cookie jars
- Handles session rotation to avoid rate limiting
- Passes JavaScript challenges through real browser execution
- Implements 30-second job staggering to stay under detection thresholds

### 3. Data Normalization at Scale

This was honestly harder than the scraping itself.

**Category problem:** 50+ different ways to say the same thing across platforms. "Edibles," "Edibles - Gummies," "Gummy," "Food," "Ingestibles" — all need to map to a single canonical category. I built a normalizer that handles exact matches, fuzzy matches, and hierarchical category resolution.

**Brand problem:** "Select," "SELECT ELITE," "Select Elite Live," and "select" are all the same brand. Across 500+ brands and 31 chains, these inconsistencies multiply fast. The brand alias engine resolves all variations to canonical brand names using a combination of exact matching, case-insensitive matching, and configurable alias tables.

**Price problem:** Some APIs return prices with tax, some without. Some show member pricing, some regular, some both with different labels. The price calculator normalizes everything to a consistent pre-tax, regular-price basis for accurate comparison.

### 4. Reliability at Scale

Running 31 scrapers on a schedule means something is always breaking — an API changes, a session expires, a rate limit hits, a page layout shifts. The system handles this through:

- **Per-chain error isolation:** One scraper failing doesn't take down the others
- **Automatic retry with exponential backoff** for transient failures
- **Job staggering:** 30-second delays between jobs to avoid thundering herd
- **Dead letter queues** for jobs that fail after retries
- **Queue monitoring dashboard** showing job status, success rates, and error logs in real-time

---

## Monorepo Structure

```
competitive-intel-platform/
├── packages/
│   ├── worker/                    # BullMQ job processing engine
│   │   ├── src/
│   │   │   ├── scraper-engine/    # API clients per platform
│   │   │   │   ├── rest-client        # Platform A integration
│   │   │   │   ├── graphql-client     # Platform B integration
│   │   │   │   ├── algolia-client     # Platform C integration
│   │   │   │   └── browser-pool       # Playwright session mgmt
│   │   │   ├── competitor-scrapers/   # 31 chain-specific configs
│   │   │   ├── job-processor.ts       # Core job orchestration
│   │   │   ├── category-normalizer.ts # Category mapping engine
│   │   │   ├── price-calculator.ts    # Price normalization
│   │   │   └── queue.ts              # BullMQ queue setup
│   │   └── cron/                  # Scheduling definitions
│   │
│   ├── scraper/                   # Base scraper framework
│   │   ├── config/                # Selectors, schedules, settings
│   │   ├── scrapers/              # Scraper modules
│   │   ├── parsers/               # Response parsing
│   │   └── storage/               # Data persistence layer
│   │
│   └── dashboard/                 # Next.js web application
│       ├── pages/                 # Dashboard views
│       └── components/            # UI components
│
├── sql/                           # PostgreSQL migrations
├── supabase/                      # Database config
├── docker/                        # Container definitions
├── scripts/                       # Utility scripts
└── .github/                       # CI/CD workflows
```

---

## Stack

| Layer | Technology |
|-------|-----------|
| **Language** | TypeScript (entire codebase) |
| **Runtime** | Node.js |
| **Queue** | BullMQ 5.x + Redis |
| **Browser Automation** | Playwright |
| **Database** | PostgreSQL via Supabase |
| **Frontend** | Next.js + React |
| **Build System** | pnpm workspaces |
| **Containerization** | Docker |
| **CI/CD** | GitHub Actions |

---

## Scale

| Metric | Value |
|--------|-------|
| **Products tracked** | 50,000+ |
| **Retail chains** | 31 |
| **States covered** | 5 |
| **API platforms reversed** | 3 (REST, GraphQL, Algolia) |
| **Chain-specific scrapers** | 31 |
| **Refresh cycle** | Every 4 hours |
| **Brand aliases managed** | 500+ |
| **Category variations normalized** | 50+ |

---

## What I Learned

- **Reverse engineering is detective work.** No documentation means you're reading network traffic, testing assumptions, and sometimes just guessing until the response makes sense. Patience matters more than cleverness.

- **Data normalization is where the real value lives.** Anyone can hit an API endpoint. Making the data from 31 different sources actually comparable and useful — that's the hard part, and that's what my team cares about.

- **Reliability beats speed.** I spent more time on error handling, retry logic, and monitoring than on the actual scraping code. A scraper that works 95% of the time is useless if you can't trust which 5% failed.

- **Build for the people who use it.** The dashboard isn't fancy. It's fast, it's clear, and it shows my team exactly what they need. That matters more than any technical architecture.

---

## Why This Repo is Private

The production codebase contains proprietary business logic, API configurations, and competitive data that belongs to my employer. This showcase documents the engineering work without exposing anything sensitive.

If you want to see my code, check out my other projects — the [Hyperliquid Trading Simulator](https://github.com/claygeo/hyperliquid-trading-sim) uses similar patterns (WebSocket data ingestion, real-time processing, state management) and is fully open source.

---

*Built by [Clayton George](https://github.com/claygeo) · [LinkedIn](https://www.linkedin.com/in/claytongeo/)*

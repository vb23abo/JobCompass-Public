# JobCompass

> An AI-powered career platform for tech professionals and graduates in the UK. Jobs come from company career sites, not aggregators. Sponsorship status comes from the Home Office register. Resume, match, apply, prepare, track, and analyse sit in one loop.

**Status:** Active development · documentation aligned with the product as of October 2026

Implementation lives in a private repository. This repository is the public product and architecture description.

---

## What JobCompass is

```
Resume → Discover → Match → Apply → Prepare → Track → Analyse
```

Each step feeds the next. The CV shapes matches. Matches feed AutoApplier. Applications open interview prep and tracking. Analytics show where the funnel leaks.

**What stays true:**

- **No aggregators.** Listings are taken from employer career pages. LinkedIn, Indeed, Glassdoor, and Welcome to the Jungle are not job sources. WTTJ is UX inspiration only.
- **UK first.** Three boards (graduate, sponsored, non-sponsored), Home Office sponsor cross-reference, UK cities and universities in the profile.
- **Hybrid architecture.** Modules that touch the live web (scraper, AutoApplier, research) are agents. Auth, matching, tracking, and analytics are deterministic pipelines.
- **Human-in-the-loop apply.** AutoApplier reads forms at runtime. It fills and waits. The user submits. CAPTCHA is a pause, never a solver.

---

## Modules

| Module | Role | Kind |
|---|---|---|
| Auth | Google or LinkedIn Log In. Identity only — no LinkedIn profile import | Pipeline |
| Profile & resume | Upload a CV or fill sections by hand. Seed → enrich → diff → summary | Pipeline + AI summary |
| Job discovery | Playwright scrape of first-party career boards. UK jobs only | Agent |
| Three boards | Graduate · sponsored · non-sponsored | Pipeline |
| Match | Semantic 60% / keyword 30% / seniority 10%. Stored per user and job | Pipeline |
| AutoApplier | Runtime form fill, HitL gates, snapshot on submit | Agent (HitL) |
| Tracker | Application FSM, optional Gmail status | Pipeline |
| Interview prep | Role guide from the posting and the profile | Pipeline + AI |
| Research | Company enrichment from public sources | Agent |
| Analytics | Funnel, boards, scores, 30-day activity | Pipeline |
| Sponsor pages | Public `/sponsors/{slug}` for licensed sponsors | Pipeline |

### Scraper, in public terms

A first visit to a careers **path** gets a how-to card (page kind, listing links, pager, optional UK filter). That card is how to walk the **page**, not how to parse a job description. Playwright collects listings. A UK posting parser then fills skills, seniority, pay, remote/hybrid, visa language, and SOC when the text supports them. Production JD parse does not call Claude.

robots.txt is honoured as `JobCompassBot`. Hosted ATS search boards are a fallback only when the employer already lists roles on their own site.

### AutoApplier gates

The agent always pauses for:

- CAPTCHA
- Field confidence below 80%
- Salary, notice period, and right-to-work
- First application to a company
- Match score below the user’s threshold (default 65%)

Default mode is fill-and-review. There is no CAPTCHA solver and no hard-coded ATS field map.

---

## Architecture

> External web → agents. Internal data → deterministic pipelines.

```
┌─────────────────────────────────────────────────────────────┐
│  CLIENT — Next.js 14 (App Router)                           │
│  Auth.js session · Tailwind + Shadcn · TanStack Query       │
└────────────────────────────┬────────────────────────────────┘
                             │ HTTPS  Bearer JWT
┌────────────────────────────▼────────────────────────────────┐
│  API — FastAPI /api/v1                                      │
│  Auth · Profile · Jobs · Applications · Prep · Analytics    │
└──────┬─────────────────┬──────────────────┬─────────────────┘
       │                 │                  │
┌──────▼──────┐  ┌───────▼────────┐  ┌─────▼──────────┐
│ PostgreSQL  │  │ Redis           │  │ Cloudflare R2  │
│ + pgvector  │  │ Celery broker   │  │ Resumes, snaps │
└─────────────┘  └───────┬────────┘  └────────────────┘
                         │ Celery
         ┌───────────────┼────────────────┐
   ┌─────▼─────┐  ┌──────▼──────┐  ┌─────▼──────┐
   │ scraper   │  │ AutoApplier │  │ AI / embed │
   └───────────┘  └─────────────┘  └────────────┘
```

---

## Stack

| Layer | Choice |
|---|---|
| Backend | Python 3.12, FastAPI, SQLAlchemy 2.0 async, Alembic |
| Frontend | Next.js 14 App Router, Auth.js v5, Tailwind, Shadcn/ui |
| Queue | Celery 5 + Redis (separate queues per workload) |
| Browser | Playwright + stealth |
| AI | Haiku for scrape how-to cards. Sonnet for summary, prep, tailor, AutoApplier. Local UK JD parser for postings |
| Embeddings | OpenAI `text-embedding-3-small` (pgvector) |
| Storage | Cloudflare R2, private, signed URLs |
| Auth | Google and LinkedIn OAuth only |
| Sponsorship | Home Office worker sponsor register |
| Company universe | Companies House snapshot |

---

## Product constraints (non-negotiable)

1. No aggregator job ingestion.
2. UK focus until an explicit product decision to expand.
3. No CAPTCHA solver.
4. Match scores from a published formula, not an LLM “decide the score”.
5. AutoApplier reads the live form. It does not ship a library of ATS DOM maps.

---

## Documentation in this repo

| File | What it is |
|---|---|
| [JobCompass_PRD.md](./JobCompass_PRD.md) | Product: users, journeys, boards, non-goals |
| [JobCompass_TRD.md](./JobCompass_TRD.md) | Technical: architecture, APIs, agents, data |

---

## License

No license. Source code is not published here.

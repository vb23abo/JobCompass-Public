# JobCompass — Technical Requirements Document

**Version:** 3.6 (public)  
**Status:** Active development, aligned with the product in October 2026  
**Depends on:** [JobCompass_PRD.md](./JobCompass_PRD.md)

This is the public technical description. Selectors, operator runbooks, environment secrets, and evaluation fixtures are not published here.

---

## 1. Purpose

Translate the PRD into architecture that can be reviewed without the private codebase: hybrid agents vs pipelines, stack, API shape, scrape/apply rules that are product commitments, and data the product stores.

---

## 2. Architecture

### 2.1 Hybrid model

> External web → agents. Internal data → deterministic pipelines.

| Module | Type | Runtime |
|---|---|---|
| Scraper | Agent | Celery `scraper` |
| AutoApplier | Agent (HitL) | Celery `applier` |
| Research | Agent | Celery `research` |
| Auth, profile, matcher, tracker, analytics | Pipeline | FastAPI `/api/v1` |
| JD parse | Pipeline | Local UK posting parser |
| Embeddings | Pipeline | OpenAI `text-embedding-3-small` |

```
CLIENT  Next.js 14 (:3000)
        Auth.js JWT · TanStack Query · Shadcn/ui
          │  Bearer access token
API     FastAPI (:8000) /api/v1
          │
     PostgreSQL 16 + pgvector     Redis (Celery + cache)     R2 (private)
          │
     Celery: scraper · applier · ai · embeddings · matcher · email · research
```

### 2.2 Why not uniform “everything is an agent”

Career sites and ATS forms change. Those paths read the page at runtime. Match, auth, and tracker do not: scores are a formula; sessions are JWTs; application status is a state machine. Mixing LLM control flow into those pipelines would make scores unauditable and auth harder to reason about.

---

## 3. Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.12, FastAPI, SQLAlchemy 2.0 async, Pydantic v2, Alembic |
| Frontend | Next.js 14 App Router, Auth.js v5, Tailwind, Shadcn/ui, TanStack Query v5 |
| Queue | Celery 5 + Redis, routed queues |
| Browser | Playwright + stealth; small concurrent context pool |
| AI | Haiku: scrape **how-to** for a careers path. Sonnet: summary, prep, tailor, AutoApplier answers. JD parse: UK rules parser in production |
| Embeddings | OpenAI `text-embedding-3-small`, 1536-d, pgvector |
| Files | Cloudflare R2, private bucket, short-lived signed URLs |
| Auth | Google + LinkedIn OAuth. 15-minute access JWT, rotating refresh family |

---

## 4. API conventions

Base path `/api/v1/`.

Success: `{ "data": { ... } }`  
Error: `{ "error": { "code": "JOBS_001", "message": "...", "request_id": "..." } }`

Namespaced codes (`AUTH_`, `RESUME_`, `JOBS_`, `APPLIER_`, `SCRAPER_`). Pagination `?page=&page_size=`. Uploads multipart, 5MB, PDF/DOCX, MIME-checked. The Next.js client sends `Authorization: Bearer` from the Auth.js session.

Rate limits exist on auth, authenticated routes, and AutoApplier enqueue.

---

## 5. Auth (public contract)

Log In is Auth.js (Google or LinkedIn) then `POST /auth/oauth/sync` on FastAPI, which upserts the user and issues tokens. The session cookie is NextAuth JWT; API calls use the access token.

There is no password register. GitHub OAuth is gone. LinkedIn does not import a profile.

Same verified email across providers is one user, via explicit merge — not automatic linking of unrelated accounts.

---

## 6. Scraper (product rules)

Jobs are collected from employer career pages the product has saved. Rules that are part of the product, not scrape folklore:

- **First-party boards first.** Prefer the employer’s own job list. Hosted ATS search (Workday, Greenhouse, Lever, Ashby, and similar) is a fallback when that first-party list does not exist. Apply links on the company page remain valid.
- **How-to, not JD extract.** A language model may write a playbook for one careers **path**: page kind, listing links, pager, optional UK filter. Playwright walks that page. Playbooks are not used to invent job IDs or to parse descriptions.
- **UK JD parser after collect.** Skills, seniority, contract, pay, remote/hybrid, visa language, and SOC come from a UK posting parser. Seniority and hybrid/remote come from the title and work-mode phrases, not from body words such as “leading a team” or “hybrid cloud”. Claude is off for this step unless an explicit comparison flag is set in a private environment.
- **UK gate.** A listing is stored only when it is about the UK (office, remote-in-UK, or a UK-filtered board with no foreign place).
- **Collect, then write.** No database writes inside the browser loop. Batch upsert at the end. Dedup by URL hash.
- **robots.txt** for `JobCompassBot`.
- **Presence.** A finished walk refreshes last-seen. Missing from a finished walk removes the job unless an application is in progress.
- **Rate limit and circuit breaker** per domain. Failures suspend that domain; they are not bypassed with block-evasion proxies.
- **Company universe.** Companies House snapshot daily; Home Office worker register daily. Licence loss → non-sponsored board. Leaving Companies House → company and scrape stop.

Embeddings and match work enqueue for **new** jobs, not the full table on every cycle.

JavaScript posting pages that return no copy on a plain GET are opened in the browser so the published description can be stored. Share-widget URLs are not jobs.

---

## 7. AutoApplier (product rules)

- Reads the live form. No library of ATS DOM maps.
- Default: fill, then wait for the user. Submit is the only send in that mode.
- Mandatory gates: CAPTCHA pause, low confidence, salary / right-to-work, first company, below match threshold, daily cap.
- Every send produces an application snapshot; success also stores a screenshot.
- Fail closed: open the employer’s apply link.
- **Non-goals:** CAPTCHA solver; proxies whose purpose is to bypass blocks.

---

## 8. Match

Stored row per `(user, job)`:

| Signal | Weight |
|---|---|
| Semantic similarity (embedding of CV vs JD) | 60% |
| Keyword overlap on required qualifications | 30% |
| Seniority | 10% |

A board request scores jobs on that page that lack a stored score. Preference and “direction” blends are sort options; they do not replace 60/30/10.

---

## 9. Data the product stores (high level)

| Area | Examples |
|---|---|
| Identity | User, OAuth subject, refresh family |
| Profile | Experience, education, skills, certificates, visa end date (optional) |
| Resume | File on R2, embedding, ATS breakdown |
| Employers | Sponsor flags, careers URLs, optional employer group sharing one careers page |
| Jobs | Title, UK location, description, board, URL hash, SOC / eligibility |
| Scores | Materialised match |
| Applications | FSM, snapshots |
| Agents | Run audit log |
| Public sponsors | Slug pages for licensed sponsors |

**Application FSM:** `saved → applied → viewed → screening → interview → offer → rejected → withdrawn`

**Boards:** `graduate` · `sponsored` · `non-sponsored`

Gmail tokens are encrypted at rest. R2 stays private.

---

## 10. Frontend (public)

Live screens use Tailwind, Shadcn, and TanStack Query. There is no mock data layer.

| Area | Route |
|---|---|
| Home after Log In | `/dashboard` |
| Resume builder | `/resume` |
| CV templates | `/resumetemplates` |
| Boards | `/jobs/graduate`, `/jobs/sponsored`, `/jobs/non-sponsored` |
| Tracker | `/applications` |
| AutoApplier | `/autoapplier` |
| Prep | `/prep` |
| Analytics | `/analytics` |
| Sponsors | `/sponsors`, `/sponsors/{slug}` |
| Log In | `/auth/login` |

The three boards are not collapsed into one Discover page.

---

## 11. Queues (names only)

`scraper`, `applier`, `ai`, `embeddings`, `matcher`, `email`, `research`, `default`.

Scheduled work (when enabled): scrape cycle, Companies House snapshot, sponsor register, apply-link recheck, eligibility classify, alert digests. The scrape schedule is off in development unless explicitly turned on.

---

## 12. Security commitments (public)

- Secrets in environment only.
- Parameterised queries.
- Short-lived access tokens.
- Private object storage.
- Encrypted Gmail tokens.
- CORS allow-list, not `*`.
- No dangerous same-email OAuth auto-link.

---

## 13. Explicit non-goals (engineering)

| Non-goal | Why |
|---|---|
| Aggregator ingest | Product constraint |
| CAPTCHA solver | Legal / ethical |
| LLM-decided match scores | Auditability |
| Hard-coded ATS maps | Forms are read at runtime |
| Harvested people graphs | Removed |
| Publishing evaluation JDs | Employer copyright; kept private |

---

## Document history (public)

| Version | Date | Change |
|---|---|---|
| 1.0 | Jun 2026 | First public TRD (scaffold) |
| 3.6 | Oct 2026 | Hybrid model, UK JD parser, playbooks as page how-to, fill-and-review AutoApplier, 60/30/10 match, OAuth-only Log In |

# JobCompass — Product Requirements Document

**Version:** 3.6 (public)  
**Status:** Active development, aligned with the product in October 2026  
**Audience:** Product · Design · Engineering · Stakeholders  
**Companion:** [JobCompass_TRD.md](./JobCompass_TRD.md)

This is the public product description. Implementation detail that would help evade site protections, copy employer job text, or expose operator runbooks is kept private.

---

## 1. Executive summary

JobCompass is an AI career platform for tech professionals and graduates in the **United Kingdom**. It takes jobs **from company career websites** — never LinkedIn, Indeed, or other aggregators — and cross-references the UK Home Office sponsor register so graduate, sponsored, and non-sponsored roles carry live sponsorship context.

```
Resume → Discover → Match → Apply → Prepare → Track → Analyse
```

| Attribute | Detail |
|---|---|
| Geography | United Kingdom |
| Primary users | International tech graduates · early-career UK tech professionals |
| Differentiator | Direct-source jobs + sponsor register + fit scores + optional AutoApplier |
| Architecture | Agents for the external web; deterministic pipelines for internal data |
| North star | Weekly active users who apply to ≥3 matched jobs and receive ≥1 employer response |

### 1.1 What is in the product

| Capability | Status |
|---|---|
| Google / LinkedIn Log In (identity only) | Shipped |
| Profile from CV upload or manual setup | Shipped |
| Resume builder (seed → enrich → diff → summary) | Shipped |
| Industry CV templates | Shipped (Word file per listed role) |
| Three job boards | Shipped |
| Public sponsor pages | Shipped (one page when several licences share a careers URL; sector hubs later) |
| Match scores (60 / 30 / 10) | Shipped |
| Scraper agent + UK JD parser | Shipped |
| AutoApplier fill-and-review | Shipped (user Submit; auto-submit locked off) |
| Tracker, prep, analytics | Shipped |
| Role-level Skilled Worker screen | Shipped as a screening signal, not immigration advice |
| Gmail connect for status mail | Shipped |

---

## 2. Vision

Every UK job seeker — especially people navigating visa sponsorship — should get navigational clarity, not another aggregator feed.

JobCompass sits above free aggregator tools (stale data, weak sponsorship signal) and below expensive recruiters. Direct-source freshness, sponsorship intelligence, and AI fit scoring are the combination.

**Non-negotiable constraints**

1. **No aggregator job sources** — employer career sites only.
2. **UK focus** — Home Office register, three boards, UK cities and universities in the profile.
3. **Hybrid architecture** — scraper, AutoApplier, and research are agents; auth, matcher, tracker, and analytics are pipelines.
4. **Resume is the profile source of truth** — OAuth supplies identity; rich profile data comes from the CV or manual entry.

Welcome to the Jungle is **UX inspiration only**. It is never a job data source.

---

## 3. Problem

1. **Discovery friction** — roles sit on hundreds of career sites.
2. **Sponsorship opacity** — the Home Office register exists; consumer job products barely use it.
3. **Resume blindness** — ATS rejection with no feedback.
4. **Volume without quality** — broad apply, low response.
5. **Preparation gaps** — interviews after screening without a role-specific brief.

---

## 4. Personas

**Priya, 23** — international MSc graduate who needs a Skilled Worker sponsor. She logs in, uploads a CV, uses the sponsored board, queues AutoApplier with HitL gates, tracks via Gmail, and opens interview prep.

**James, 26** — UK citizen, two years in industry. He uses the resume builder, graduate or non-sponsored boards, and analytics to see why responses are thin.

---

## 5. Architecture philosophy

> External web → autonomous agents. Internal data → deterministic pipelines.

| Module | Type |
|---|---|
| Scraper | Agent |
| AutoApplier | Agent with mandatory human gates |
| Research | Agent |
| Auth, profile, matcher, tracker, analytics | Deterministic |

**AutoApplier always pauses on:** CAPTCHA; confidence below 80%; salary / right-to-work; first application to a company; match below threshold. **Never** a CAPTCHA solver.

Match scores are a published weighted formula. An LLM does not “decide” the score.

---

## 6. Journeys

### Log In and profile

- Log In is Google or LinkedIn. There is no email/password register.
- After Log In the user lands on the dashboard.
- First-run: upload a CV or fill the profile by hand. LinkedIn is not imported as a CV.

### Resume

Unified builder at `/resume`: seed → enrich → diff → summary. The uploaded CV remains authoritative for match and AutoApplier.

### Discovery

Three boards stay browsable: **graduate**, **sponsored**, **non-sponsored**. Preferences rank and filter; they do not hide a board. Cards show match %, a short “why you fit”, a sponsorship badge, and **Salary not listed** when pay is absent.

### Apply, track, prepare, analyse

AutoApplier default is fill, then wait. The user submits. Each successful send stores a snapshot. Tracker FSM: `saved → applied → viewed → screening → interview → offer → rejected → withdrawn`. Prep is a role guide from the posting. Analytics cover funnel, boards, and activity over time.

---

## 7. Feature requirements (public)

### Boards

| Board | Meaning |
|---|---|
| Graduate | Entry-level and graduate schemes |
| Sponsored | Home Office licensed sponsors |
| Non-sponsored | Employers not on that register |

Company mini-profiles show register status, scrape freshness, and open roles — not employer brand studio content.

### Scraper

Jobs come from saved first-party careers URLs. Dedup is by URL hash. Per-domain rate limits and a circuit breaker apply. Embeddings run for **new** jobs, not the whole table on every cycle. UK location is required to store a listing.

The company list is the Companies House snapshot. Sponsor status is a flag from the Home Office worker register. Losing a licence moves jobs to the non-sponsored board; leaving Companies House removes the company.

**Non-goal:** LinkedIn, Indeed, Glassdoor, or any aggregator ingest.

### Match

Stored per user and job. Semantic similarity, keyword overlap on required qualifications, and seniority. Opening a board scores jobs on that page that have no stored score. A scrape does not rescore every user.

### AutoApplier

Reads the live form. No hard-coded ATS maps. Daily cap and match threshold are enforced on the server. Default: fill and wait. Optional auto-submit stays off. Fail closed to a manual apply link.

### Tracker, prep, analytics

Applications keep title, company, location, and match after a job row is gone. Prep uses Claude with a template fallback. Analytics include summary, funnel, and time series.

---

## 8. Identity

| Method | Role |
|---|---|
| Google OAuth | Log In |
| LinkedIn OIDC | Log In — no profile import |
| Gmail OAuth | Optional mail for tracker — not login |
| Email + password | Removed |

Progressive trust: disposable-email block, bound OAuth subject, phone OTP before AutoApplier queue, optional ID check for a higher cap. Legal name + date of birth is **not** used as a uniqueness signal.

Same verified email across Google and LinkedIn belongs to one account, via an explicit merge — not silent account linking.

---

## 9. Shipped vs later

**Shipped:** direct scrape and three boards; resume builder and ATS score; materialised match; AutoApplier with HitL; tracker, prep, analytics; Skilled Worker going-rate screen; OAuth-only Log In; public sponsor pages; privacy and terms pages (counsel wording still required).

**Open:** formal legal / ToS counsel sign-off before a UK marketing push; production scrape legal-approval flag after that review.

**Later:** EU / Canada; employer-side tooling; native mobile.

---

## 10. Non-goals

| Non-goal | Why |
|---|---|
| Aggregator job ingest | Product constraint |
| LinkedIn profile import | Resume or manual only |
| Fully autonomous apply | Legal and ethical; CAPTCHA |
| LLM-decided match scores | Formula stays auditable |
| CAPTCHA solver | Explicit non-goal |
| Harvested hiring-manager directories | Removed |
| GitHub OAuth / password login | Removed |
| DOB + name hash as identity | GDPR risk |

---

## 11. Success metrics (MVP)

| Metric | Target |
|---|---|
| North star | ≥30% WAU apply to ≥3 jobs/week |
| Sponsor board | ≥40% of international users on Sponsored |
| Match precision | ≥60% of saves above threshold |
| AutoApplier | ≥50% of queued items complete without abort |
| Freshness | Median job age under 72h on priority companies |

---

## 12. Landscape

| | Sponsorship | Direct scrape | Apply assist | UK graduate |
|---|---|---|---|---|
| LinkedIn | Unreliable | No | No | Partial |
| Indeed | None | No | No | Partial |
| Gradcracker | Limited | No | No | Strong |
| JobCompass | Register + role-level going-rate screen | Yes | HitL + eligibility gate | Yes |

---

## Document history (public)

| Version | Date | Change |
|---|---|---|
| 1.0 | Jun 2026 | First public PRD (scaffold) |
| 3.6 | Oct 2026 | Brought in line with the shipped product: UK JD parser, playbook-as-how-to, fill-and-review AutoApplier, three boards, OAuth-only Log In |

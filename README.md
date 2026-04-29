# Job Scraping Pipeline

Automated daily pipeline that scrapes Israeli job boards, scores matches against a candidate profile, generates tailored CVs using an LLM failover chain, finds recruiter contact details, and sends cold outreach — all without manual intervention.

---

## The Problem

Searching for a senior engineering role across 6 Israeli job boards daily is 2–3 hours of manual work. Most postings are duplicated across platforms. CVs need to be tailored per role. Recruiter emails are hard to find. Sending outreach at scale is repetitive.

---

## What It Does

| Step | What happens |
|------|--------------|
| Scrape | Pulls job listings from LinkedIn, Indeed, AllJobs, Drushim, Wellfound, and Glassdoor |
| Deduplicate | Cross-source fingerprinting removes duplicates across boards |
| Score | Weighted match scoring (skills, title, experience, location) ranks every job 0–100% |
| Generate CV | Tailored CV + cover letter generated per high/medium-match job |
| Find recruiter | Email discovery via Hunter.io → Snov.io → Apollo.io cascade |
| Outreach | Cold email sent via Resend; dedup log prevents re-contacting |
| Report | Excel + CV attachments emailed at end of each run |

---

## Technology

- **Python 3.11** — core language
- **Selenium + undetected-chromedriver** — browser automation with anti-bot mitigations
- **LLM failover chain** — Gemini → Groq → Cerebras → Mistral → OpenRouter → Anthropic; skips dead providers per session
- **ATS coverage check** — CV rejected and regenerated if keyword coverage < 75%
- **PostgreSQL** — optional run history, job dedup, scraper stats
- **Resend** — transactional email for reports and outreach
- **Hunter.io / Snov.io / Apollo.io** — recruiter email discovery

---

## Design Principles

- **Fail-safe scraping** — each scraper runs in isolation; one failure doesn't abort the pipeline
- **Per-run dedup** — `seen_jobs.txt` + fingerprint file persist across runs; jobs aren't reprocessed daily
- **Dynamic model selection** — LLM providers queried at runtime for their best available model; no hardcoded model names
- **Graceful degradation** — if all jobs are duplicates, a stats-only email is sent and the pipeline exits cleanly

---

## Israeli job boards covered

| Board | Language | Notes |
|-------|----------|-------|
| LinkedIn | English | Login required |
| Indeed | English | Cloudflare-hardened |
| AllJobs | Hebrew | ASP.NET pagination |
| Drushim | Hebrew | Nuxt.js JSON extraction |
| Wellfound | English | Optional login |
| Glassdoor | English | Salary data appended |

---

*The implementation is private. This page covers the design.*

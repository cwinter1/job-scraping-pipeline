# Job Scraping Pipeline

Automated daily pipeline that scrapes Israeli job boards, scores matches against a candidate profile, generates tailored CVs using an LLM failover chain, finds recruiter contact details, and sends cold outreach — all without manual intervention.

But it's also a job prep tool. Every run produces structured data about the market: which skills are trending, which companies are hiring, which titles are being used. That data feeds directly into interview preparation.

---

## The Problem

Searching for a senior engineering role across 6 Israeli job boards daily is 2–3 hours of manual work. Most postings are duplicated across platforms. CVs need to be tailored per role to pass ATS filters. Recruiter emails are hard to find. Sending outreach at scale is repetitive.

And even when you get interviews, you're often going in cold — no data on what the company is actually hiring for, what skills they weight, or how they describe the role internally.

This pipeline solves both problems.

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
| Trend analysis | TF-IDF skill frequency analysis across all scraped JDs, appended to daily email |

---

## Interview Prep Layer

The pipeline does more than find jobs. It builds a picture of what the market is actually asking for — and that picture is directly useful when preparing for interviews.

**Skill frequency analysis**
Each run extracts skills from every job description using TF-IDF scoring. Skills in the top quartile of the candidate's known stack are flagged. Emerging terms — skills appearing frequently but not yet in the profile — are surfaced separately. After a week of runs, you have a ranked list of what Israeli companies are actually asking for right now, not what was on a blog post six months ago.

**Per-company context**
Before an interview, you can query the pipeline's history for that specific company: every role they've posted, the language they use in JDs, the stack mentioned, the seniority signals. Going into an interview knowing that a company consistently mentions Kafka and Airflow in their JDs — and preparing accordingly — is a material advantage.

**Title normalisation**
The same role gets posted as "Data Engineer", "Senior DE", "Data Platform Engineer", "Analytics Engineer", and "Data Infrastructure Engineer" depending on the company. The pipeline maps these to canonical titles, so you can see how different companies describe the same work — and calibrate how to present your experience for each.

**ATS reverse-engineering**
Every generated CV is scored for keyword coverage against the target JD. CVs below 75% coverage are rejected and regenerated. Running this in reverse — reading which keywords the ATS check flags as missing — tells you exactly which terms to use when describing your experience in interviews.

---

## Skills & Commands

The pipeline is designed to be operated from the command line. Core usage:

```bash
# Run all 6 sources headless
python main.py --headless

# Israeli boards only (faster, no login required)
python main.py --sources alljobs drushim wellfound

# Single source for debugging
python main.py --sources linkedin
python main.py --sources indeed

# Include remote jobs in addition to Israel
python main.py --headless --remote

# Re-run CV generation on the latest results without scraping
python utils/cv_generator.py

# Preview recruiter outreach without sending
python run_outreach.py 20260429 --dry-run

# Send outreach for a specific run date
python run_outreach.py 20260429

# Check all API keys are live
python test_apis.py

# Generate a one-off tailored CV for a specific job
python generate_cv_single.py
```

The pipeline also supports PostgreSQL persistence (`DATABASE_URL` env var) for tracking run history, match scores, and CV generation results across sessions.

---

## Technology

- **Python 3.11** — core language
- **Selenium + undetected-chromedriver** — browser automation with anti-bot mitigations
- **LLM failover chain** — Gemini → Groq → Cerebras → Mistral → OpenRouter → Anthropic; skips dead providers per session
- **ATS coverage check** — CV rejected and regenerated if keyword coverage < 75%
- **TF-IDF (sklearn)** — skill frequency scoring across scraped job descriptions
- **spaCy** — lemmatisation for skill normalisation before trend analysis
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

## Israeli Job Boards Covered

| Board | Language | Notes |
|-------|----------|-------|
| LinkedIn | English | Login required |
| Indeed | English | Cloudflare-hardened |
| AllJobs | Hebrew | ASP.NET pagination |
| Drushim | Hebrew | Nuxt.js JSON extraction |
| Wellfound | English | Optional login |
| Glassdoor | English | Salary data appended |

---

*The implementation is private. This page covers the design and methodology.*

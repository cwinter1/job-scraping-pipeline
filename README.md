# Job Scraping Pipeline

Automated daily pipeline that scrapes Israeli job boards, scores matches against a candidate profile, generates tailored CVs using an LLM failover chain, finds recruiter contact details, and sends cold outreach — all without manual intervention.

But it's also a job prep tool. Every run produces structured data about the market: which skills are trending, which companies are hiring, which titles are being used. That data feeds directly into interview preparation.

---

## The Problem

Searching for a senior engineering role across 6 Israeli job boards daily is 2–3 hours of manual work. Most postings are duplicated across platforms. CVs need to be tailored per role to pass ATS filters. Recruiter emails are hard to find. Sending outreach at scale is repetitive.

And even when you get interviews, you're often going in cold — no data on what the company is actually hiring for, what skills they weight, or how they describe the role internally.

This pipeline solves both problems.

---

## Architecture

```
main.py  (orchestrator)
│
├── Step 0  API health check — Gemini, Groq, Cerebras, Mistral, OpenRouter,
│           Anthropic, Hunter.io, Resend
│
├── Step 1  Scrape (6 sources, isolated per scraper)
│           ├── linkedin_scraper.py   Selenium DOM, English keywords
│           ├── indeed_scraper.py     undetected-chromedriver, Cloudflare bypass
│           ├── alljobs_scraper.py    Selenium, ASP.NET pagination, Hebrew keywords
│           ├── drushim_scraper.py    Nuxt.js window.__NUXT__ JSON extraction
│           ├── wellfound_scraper.py  Next.js __NEXT_DATA__ JSON extraction
│           └── glassdoor_scraper.py  undetected-chromedriver, salary appended
│           → dedup by URL + company/title fingerprint → new_jobs.json
│
├── Step 2  Score (job_matcher.py)
│           Skills 40% | Title 20% | Experience 20% | Location 10% | Bonus 10%
│           → matched_jobs.json
│
├── Step 3  Report (report_generator.py)
│           → Excel + CSV output, email draft
│
├── Step 4  CV Generation (cv_generator.py)
│           LLM failover: Gemini → Groq → Cerebras → Mistral → OpenRouter → Anthropic
│           ATS coverage check per CV — reject + regenerate if < 75%
│           → cv_YYYYMMDD/ folder
│
├── Step 5  Recruiter discovery (recruiter_finder.py)
│           Hunter.io → Snov.io → Apollo.io
│           + cold outreach via Resend (cold_outreach.py)
│
├── Step 6  Email results
│           Excel + CVs + cover letters + trend summary block
│
└── analyze_trends.py  (runs every execution)
            TF-IDF scoring across all scraped JDs
            spaCy lemmatisation + SYNONYM_MAP normalisation
            Top-quartile skill markers + emerging terms block
            → appended to Step 6 email
```

**Storage**
```
new_jobs.json              raw scraped jobs per run
matched_jobs.json          scored jobs per run
seen_jobs.txt              URL dedup across runs
seen_fingerprints.txt      company+title dedup across runs
logs/cv_cache.json         cached CV outputs — avoids regenerating same job
logs/cv_failures.csv       failed CV attempts with error reason
logs/recruiter_cache.json  found/tried recruiter emails per company
logs/outreach_log.csv      sent outreach log — prevents duplicates
```

**Database (PostgreSQL — optional)**
```
job_runs        one row per pipeline execution (status, counts, timing)
jobs            all scraped jobs, deduped by URL (first_seen / last_seen)
matches         score + keyword breakdown per job per run
cv_generations  model_used, template_type, cv_path per job per run
scraper_stats   per-(source, keyword) job counts per run
```

Docker runs PostgreSQL in WSL2 with mirrored networking so `localhost:5432` is reachable from Windows. The pipeline writes to the DB after each scraper completes — not at the end — so a mid-run crash still saves partial stats.

---

## Trend Analysis (`analyze_trends.py`)

Runs on every pipeline execution alongside the main steps. Analyses all scraped job descriptions to surface what the market is actually asking for right now.

**How it works:**
1. All JD text is lemmatised using spaCy, then normalised through `SYNONYM_MAP` (k8s→kubernetes, js→javascript, pg→postgresql, etc.)
2. TF-IDF scores are computed across the full corpus using sklearn
3. Skills in the candidate's known stack that score in the top quartile are marked with `*tfidf`
4. Emerging terms — high-frequency skills not yet in the profile — are listed separately
5. The full summary is appended to the daily results email

This turns the pipeline from a job-finder into a live market signal. After a week of runs you have ranked, normalised, deduplicated data on which skills Israeli companies are actually requesting — not what was trending on a blog post six months ago.

---

## Failsafe Design

Every failure mode has a defined fallback:

| Scenario | Behaviour |
|----------|-----------|
| Scraper raises exception | Logged, skipped; remaining scrapers continue |
| All scrapers return 0 new jobs | Stats-only email sent; pipeline exits cleanly |
| LLM provider rate-limited or down | Skipped for session; next provider in chain used |
| CV fails ATS check twice | Logged to `cv_failures.csv`; job excluded from results |
| Recruiter email not found | All 3 APIs tried; result cached so they're not retried next run |
| Excel file locked (open in Excel) | Try-except catches lock; pipeline continues to email step |
| PostgreSQL unavailable | DB writes skipped; file-based outputs unaffected |
| `undetected_chromedriver` destructor error | Caught in `close()`; `self.driver = None` prevents double-free |

Each scraper writes its stats to the DB independently. If Steps 4–6 crash, Step 1–2 data is already persisted.

---

## QA Automation

The pipeline ships with automated QA across multiple layers:

**API health check (Step 0 / `test_apis.py`)**
Validates all external API keys before the pipeline does any work: Gemini, Groq, Cerebras, Snov.io, Apollo.io, Resend, PostgreSQL. Any dead key is reported before a single browser opens.

**Scraper interface tests**
Each scraper is validated against a fixed contract: `setup_driver()`, `search_jobs()`, `close()`, `self.jobs` list, and all 9 required job dict fields (`id`, `title`, `company`, `location`, `description`, `url`, `posted_date`, `source`, `scraped_at`). A scraper that returns malformed output fails the check before it touches the dedup layer.

**CV content tests (`tests/test_cv_templates_content.py`)**
13 tests covering:
- Verified metrics present in generated CVs (23% conversion, 6-8h→4h ETL, 50+ stores)
- No fabricated achievements or unverifiable percentages
- Banned words not present (`seasoned`, `passionate`, `leverage`, `proven track record`)
- ATS coverage threshold enforced

**Clarity check (post-generation)**
Every generated CV runs through `_check_about_me_clarity()`: banned word scan + AI rates clarity 1–5 for a non-native English reader. Score < 4 triggers an automatic retry with flagged issues listed in the prompt.

**Debug output**
Each scraper saves `output/SITE_debug.html` when it returns 0 results — giving a snapshot of exactly what the browser saw.

---

## Interview Prep Layer

**Skill frequency analysis**
TF-IDF scoring across all scraped JDs flags top-quartile known skills and surfaces emerging terms. After a week of runs, you have a ranked list of what Israeli companies are actually asking for right now.

**Per-company context**
Query the pipeline's DB history for a specific company: every role they've posted, the language they use in JDs, the stack mentioned, the seniority signals. Going into an interview knowing what a company consistently asks for — and preparing accordingly — is a material advantage.

**ATS reverse-engineering**
Every CV is scored for keyword coverage against the target JD. Reading which terms the ATS check flags as missing tells you exactly which words to use when describing your experience in interviews.

---

## Skills & Commands

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

---

## Technology

- **Python 3.11** — core language
- **Selenium + undetected-chromedriver** — browser automation with anti-bot mitigations
- **LLM failover chain** — Gemini → Groq → Cerebras → Mistral → OpenRouter → Anthropic
- **ATS coverage check** — CV rejected and regenerated if keyword coverage < 75%
- **TF-IDF (sklearn) + spaCy** — skill frequency scoring and lemmatisation across JDs
- **PostgreSQL (Docker / WSL2)** — run history, job dedup, match scores, scraper stats
- **Resend** — transactional email for reports and outreach
- **Hunter.io / Snov.io / Apollo.io** — recruiter email discovery

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

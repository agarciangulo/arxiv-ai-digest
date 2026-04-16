# ArXiv AI Digest

A daily automated digest of the most relevant AI research on arXiv — curated to a personal interest profile, summarized in plain language, delivered by email every morning.

**Status:** shipped and running. A scheduled GitHub Action runs every weekday at 6 AM ET, reads the latest cs.AI listings, and emails a digest.

## The problem

arXiv publishes 150–200 AI papers every weekday. Most are narrow; a few matter; none come sorted. Reading the list is a chore, and the summaries you get from a tweet feed or Hacker News filter by someone else's interests, not yours.

This agent reads the whole list for you, ranks against a profile *you* write (what you care about, what you don't, which labs get a tiebreaker), and deep-dives the top five so you can actually understand them.

## What it does

Every morning, Tuesday through Saturday:

1. **Pulls ~200 papers** from arXiv's cs.AI daily listing.
2. **Ranks them against your profile** — a JSON file describing your interests, deprioritizations, preferred venues, and a "wildcard" rule for surfacing novel paradigms.
3. **Deep-dives the top 5** — downloads the full PDFs, asks Claude for 1,000–2,000 word accessible summaries of each. Not the abstract restated — actual lay-audience explanations of what the paper does and why it matters.
4. **Generates short blurbs for the next 5** — one paragraph each, explaining why they made the list.
5. **Composes an HTML email** and sends it via Gmail SMTP to the subscriber list.
6. **Handles quiet days** — on weekends or slow listing days, sends a short "quiet day" note instead of forcing a digest.
7. **Fails loudly** — any stage error triggers an error email rather than a silent skip.

## Architecture

```
arXiv API  ─►  Stage 1: Ranker          (Haiku — all titles + abstracts → top 10)
                   │
                   ▼
               Stage 2: Deep Analyzer   (Sonnet — full PDFs of top 5 → long summaries)
                   │
                   ▼
               Blurb Generator          (Haiku — top 6-10 → one-paragraph blurbs)
                   │
                   ▼
               Email Composer           (Jinja2 HTML template)
                   │
                   ▼
               Gmail SMTP  →  subscribers
```

No databases, no servers, no queues. Stateless pipeline, idempotent re-runs, linear Python. The whole thing runs inside a 15-minute GitHub Actions timeout.

Two-stage ranking is deliberate: a cheap model reads everything, an expensive model reads only the shortlist. Keeps the daily compute cost low without sacrificing depth on the papers that matter.

## Why the planning effort

Before writing any code, I wrote:

- [SCOPE.md](docs/SCOPE.md) — objective, deliverables, constraints
- [ARCHITECTURE.md](docs/ARCHITECTURE.md) — components, flows, prompt specs
- [TECH_STACK.md](docs/TECH_STACK.md) — every library, why it was chosen
- [EXECUTION_PLAN.md](docs/EXECUTION_PLAN.md) — phase-by-phase build sequence with verification checkpoints

The point wasn't rigor for rigor's sake. The interesting architectural choices — two-stage ranking, profile-driven relevance, stateless by design, failing loud — are only interesting if you commit to them before implementation, because they constrain the code you write. Writing them down first let me build straight through without doubling back.

Those docs are still in the repo as the authoritative design reference. The code matches them.

## Tech stack

- **Python 3.12**
- **GitHub Actions** — daily cron (no servers to manage)
- **Anthropic Claude** — Haiku for ranking/blurbs, Sonnet for deep analysis
- **arXiv API** via `feedparser`
- **PyPDF2** for full-paper extraction
- **Jinja2** for email templates
- **Gmail SMTP** for delivery

## Running it yourself

1. **Fork the repo.**
2. **Edit [`config/user_profile.json`](config/user_profile.json)** — this is the file that tells the LLM what "relevant" means to you. Fill in your role, interests, deprioritized topics, and preferred sources. The placeholder version shows the structure.
3. **Edit [`config/subscribers.json`](config/subscribers.json)** — add email addresses to receive the digest.
4. **Add GitHub secrets** in your fork:
   - `ANTHROPIC_API_KEY` — your Anthropic key
   - `GMAIL_ADDRESS` — the sending account
   - `GMAIL_APP_PASSWORD` — a Gmail app password (not your account password)
5. **Enable the GitHub Actions workflow.** It will run on the daily schedule.

To run locally first:

```bash
pip install -r requirements.txt
cp .env.example .env   # fill in the same secrets
DRY_RUN=true python src/main.py   # runs pipeline without sending
```

## Repository structure

```
src/
  collector.py         arXiv API + paper metadata
  ranker.py            Stage 1 ranking (titles + abstracts)
  pdf_extractor.py     PDF download and text extraction
  analyzer.py          Stage 2 deep dives
  blurb_generator.py   Short blurbs for ranks 6-10
  email_composer.py    HTML email assembly
  email_sender.py      Gmail SMTP delivery
  main.py              Pipeline orchestrator
config/
  user_profile.json    Interest profile (customize this)
  subscribers.json     Recipient list
templates/
  digest_email.html    Jinja2 HTML email
  quiet_day_email.html Fallback for slow days
tests/                 Unit + extraction spot-checks
.github/workflows/
  daily_digest.yml     Cron schedule and job definition
docs/                  Design artifacts
```

## License

MIT — see [LICENSE](LICENSE).

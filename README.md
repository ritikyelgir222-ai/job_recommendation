# Generative AI Powered Job Recommender System (JobMatch AI)

Pulls **live job postings** from LinkedIn, Indeed and Monster (through [Apify](https://apify.com) actors), extracts the skills each posting asks for, and ranks the jobs against your candidate profile. UI: single-page app in the "Gen-Z MCP" style; backend: FastAPI + SQLite.

> 📘 Full documentation (architecture, API reference, scoring, troubleshooting, changelog): **[docs/PROJECT_DOCUMENTATION.md](docs/PROJECT_DOCUMENTATION.md)**

## Quick start

Requires **Python 3.10+**.

```bash
python -m venv .venv
# Windows: .venv\Scripts\activate      macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env        # Windows: copy .env.example .env
# edit .env and set APIFY_TOKEN

uvicorn backend.main:app --reload
```

Open **http://127.0.0.1:8000/app/** → *Job Search Engine* → enter keywords/location → **Fetch Live Jobs**.

Without a token the app still works on built-in **sample** jobs (clearly labelled "SAMPLE DATA").

## Secrets

* The Apify token lives only in `.env` (git-ignored). **Never put it in `index.html` or commit it.**
* The token that was pasted in `api.txt` has been exposed in plain text. **Rotate it** at https://console.apify.com/settings/integrations and put the new one in `.env`.
* `.env` and `backend/data/` are git-ignored; never zip or share them.
* `.env` also understands the original variable names from `api.txt` (`APIFY_LINKEDIN_API`, `INDEED_API_KEY`, `MONSTER_API_KEY`) if you prefer a token per board; a full `...?token=` URL is accepted too.

## Project structure

```text
├── .env.example
├── docs/PROJECT_DOCUMENTATION.md
├── tests/                       # pytest suite (37 tests, offline)
├── frontend/
│   ├── index.html               # UI (live-fetch panel in the Jobs tab)
│   └── vendor/lucide.min.js     # icons, bundled so the UI works offline
├── backend/
│   ├── main.py                  # API routes + background refresh runner
│   ├── config.py                # env/.env loading, token lookup, secret scrubbing
│   ├── db.py                    # SQLite: jobs, applications, profile, feedback
│   ├── skills.py                # skill vocabulary, experience + work-type detection
│   ├── matching.py              # candidate <-> job scoring
│   ├── seed.py                  # sample jobs (offline fallback)
│   └── sources/
│       ├── base.py              # Apify REST runner + field normalisation
│       └── providers.py         # LinkedIn / Indeed / Monster actor inputs
└── requirements.txt
```

## How it works

1. **Fetch**: `POST /api/jobs/refresh` starts one background thread per board. Each calls the Apify `run-sync-get-dataset-items` endpoint for its actor (LinkedIn `hKByXkMQaC5Qt9UMN`, Indeed `TrtlecxAsNRbKl1na`, Monster `9d4mn9SVh5WvvtLii`). A failing board never blocks the others.
2. **Normalise**: raw actor records are mapped to the UI's job shape (title, company, location, salary, posted date, work type, experience bucket, description). Skills are extracted from the posting text.
3. **Score**: `match = 80 × skill coverage + title-fit (12) + location-fit (8) ± feedback adjustment`, clamped to 0–100. Thumbs up/down in *AI Feed* nudges future scores.
4. **Persist**: jobs, applications, profile and feedback are stored in `backend/data/jobmatch.db`.

## API

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/jobs/refresh` | Start a live fetch → `run_id` |
| GET | `/api/jobs/refresh/{run_id}` | Per-board progress / errors |
| GET | `/api/jobs` | Stored jobs scored for your profile (`search`, `work_type`, `experience`, `source`) |
| DELETE | `/api/jobs` | Clear stored live jobs |
| GET | `/api/sources` | Which boards have a token (never returns tokens) |
| GET | `/api/sources/{source}/sample` | Dev aid: raw + normalised sample (3 jobs) from one actor |
| POST | `/api/recommend` | Rank jobs for a skill list / role / location |
| GET/PUT | `/api/profile` | Candidate profile |
| GET/POST/PATCH | `/api/applications` | Application tracker |
| POST | `/api/feedback` | Thumbs up/down |
| POST | `/api/mcp/pipeline` | Runs the scoring pipeline and reports each step |

Interactive docs: http://127.0.0.1:8000/docs

## Things to know

* **Verify the field mapping once.** Each Apify actor has its own output format and the normaliser tolerates many field-name variants (including Indeed's `datePublished`, `baseSalary` and structured `location`), but the live actors have not been run against real credits in this repo's test suite. Call `GET /api/sources/linkedin/sample` (and `indeed`, `monster`) once; if something such as salary or location comes back empty, adjust the alias lists in `backend/sources/base.py` (`normalize_item`).
* **Cost & time.** Every fetch consumes Apify credits and takes ~30–120 s per board. The synchronous endpoint is capped at ~300 s, so keep "Results per board" modest.
* **Coverage.** The Monster actor only supports the **United States**; for any other country the API skips it (and the UI greys it out). LinkedIn/Indeed support India (`DEFAULT_COUNTRY=in`).
* **Security.** Job text is scraped from third parties; the frontend HTML-escapes every field and only allows `http(s)` links.
* **Scoring is rule-based**, not a trained or generative model. Natural next steps: embeddings for semantic matching, an LLM for resume parsing and match explanations, and a real MCP server for the "tools" tab (currently a simulation).
* Scraping job sites may be restricted by their terms of service; check that your use complies.

## Tests

```bash
pip install -r requirements.txt -r requirements-dev.txt
pytest
```

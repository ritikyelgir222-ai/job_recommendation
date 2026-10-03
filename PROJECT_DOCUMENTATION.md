# JobMatch AI — Project Documentation

**Generative AI Powered Job Recommender System** · version 3.0.0

JobMatch AI pulls live job postings from LinkedIn, Indeed and Monster (through [Apify](https://apify.com) actors), extracts the skills each posting asks for, and ranks the postings against a candidate profile. A single-page web UI sits on top of a FastAPI + SQLite backend.

---

## Table of contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Repository layout](#3-repository-layout)
4. [Getting started](#4-getting-started)
5. [Configuration](#5-configuration)
6. [How it works](#6-how-it-works)
7. [Scoring algorithm](#7-scoring-algorithm)
8. [Data model](#8-data-model)
9. [REST API reference](#9-rest-api-reference)
10. [Frontend guide](#10-frontend-guide)
11. [Job-board sources (Apify actors)](#11-job-board-sources-apify-actors)
12. [Testing](#12-testing)
13. [Security notes](#13-security-notes)
14. [Troubleshooting](#14-troubleshooting)
15. [Known limitations and roadmap](#15-known-limitations-and-roadmap)
16. [Changelog — fixes in this revision](#16-changelog--fixes-in-this-revision)

---

## 1. Overview

| | |
|---|---|
| **Goal** | Show a candidate the live jobs that best fit their skills, and track applications. |
| **Backend** | Python 3.10+, FastAPI, Pydantic v2, httpx, SQLite (standard library) |
| **Frontend** | One static `index.html` (vanilla JS, no build step), icons from a vendored copy of Lucide |
| **Data source** | Apify actors for LinkedIn, Indeed and Monster; built-in sample jobs as an offline fallback |
| **Matching** | Rule-based: skill coverage + title fit + location fit + thumbs-up/down feedback |
| **Persistence** | `backend/data/jobmatch.db` (SQLite, created on first use) |

Two data modes exist and the UI labels them clearly:

* **SAMPLE** – five built-in jobs, shown until the first successful live fetch.
* **LIVE** – jobs fetched from Apify and stored in SQLite. They replace the sample set.

> The "Gen-Z MCP" tab and the `/api/mcp/*` endpoints are a **simulation** of an MCP-style tool pipeline. They run the real scoring code and report each step, but no external MCP server is involved.

---

## 2. Architecture

```text
┌────────────────────────────┐        HTTP / JSON         ┌────────────────────────────────────────┐
│  Browser                   │ ─────────────────────────▶ │  FastAPI app  (backend/main.py)        │
│  frontend/index.html       │ ◀───────────────────────── │                                        │
│  (served at /app/)         │                            │  routes ─▶ matching.py  (scoring)      │
└────────────────────────────┘                            │         ─▶ db.py        (SQLite)       │
                                                          │         ─▶ sources/     (Apify)        │
                                                          └──────────────┬─────────────────────────┘
                                                                         │ one thread per board
                                                                         ▼
                                                  ┌──────────────────────────────────────────┐
                                                  │ Apify REST API  run-sync-get-dataset-items│
                                                  │  LinkedIn · Indeed · Monster actors       │
                                                  └──────────────────────────────────────────┘
```

**Request flow for a live fetch**

1. The UI calls `POST /api/jobs/refresh`; the backend validates the request, skips boards that have no token or do not support the chosen country, and starts a background thread. It immediately returns a `run_id`.
2. The background thread starts one worker thread per board. Each worker calls its Apify actor, normalises the raw records, and upserts them into SQLite. A failure in one board never affects the others.
3. The UI polls `GET /api/jobs/refresh/{run_id}` every 2.5 s until the run is `done` or `error`.
4. The UI then reloads `/api/jobs` and `/api/recommend` (scored for the current profile) and re-renders.

---

## 3. Repository layout

```text
.
├── .env.example               # template for local secrets (copy to .env)
├── .gitignore
├── pytest.ini
├── requirements.txt           # runtime dependencies (includes backend/requirements.txt)
├── requirements-dev.txt       # test dependencies
├── run.sh / run_windows.bat   # create venv, install, start the server
├── README.md                  # quick start
├── docs/
│   └── PROJECT_DOCUMENTATION.md   # this file
├── frontend/
│   ├── index.html             # whole UI
│   └── vendor/                # lucide.min.js (icons) + its licence
├── backend/
│   ├── main.py                # FastAPI app, routes, background refresh runner
│   ├── config.py              # .env loading, per-board token lookup, secret scrubbing
│   ├── db.py                  # SQLite access (jobs, applications, key/value)
│   ├── skills.py              # skill vocabulary + skill/experience/work-type extraction
│   ├── matching.py            # candidate ↔ job scoring
│   ├── seed.py                # sample jobs and default profile
│   ├── sources/
│   │   ├── base.py            # Apify runner + defensive field normalisation
│   │   └── providers.py       # actor ids and inputs for LinkedIn / Indeed / Monster
│   └── data/                  # jobmatch.db lives here (git-ignored)
└── tests/                     # pytest suite (37 tests, no network needed)
```

---

## 4. Getting started

### Prerequisites

* Python **3.10 or newer** (the code uses `X | None` and `tuple[...]` annotations)
* An Apify account and API token — only needed for **live** jobs. Without it the app runs on sample data.

### Quick start

macOS / Linux:

```bash
./run.sh
```

Windows: double-click `run_windows.bat`.

Both scripts create `.venv`, install `requirements.txt`, and start `uvicorn` on <http://127.0.0.1:8000>.

Manual setup:

```bash
python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                 # Windows: copy .env.example .env
# edit .env and set APIFY_TOKEN
uvicorn backend.main:app --reload
```

### URLs

| URL | What |
|---|---|
| `http://127.0.0.1:8000/app/` | The web app |
| `http://127.0.0.1:8000/docs` | Interactive Swagger UI |
| `http://127.0.0.1:8000/api/health` | Health check (mode, number of jobs, configured boards) |

### First live fetch

1. Open the app → **Job Search Engine**.
2. Enter keywords and location, choose a country and boards, click **Fetch Live Jobs**.
3. Wait 30–120 s per board. The status line shows progress per board; the tag in the card turns to **LIVE DATA**.

---

## 5. Configuration

All settings are environment variables, normally set in `.env` at the project root (loaded by `python-dotenv`; real environment variables win over `.env`).

| Variable | Default | Meaning |
|---|---|---|
| `APIFY_TOKEN` | – | Shared Apify token used for every board. |
| `APIFY_LINKEDIN_TOKEN` / `APIFY_LINKEDIN_API` | – | Token for LinkedIn only (overrides `APIFY_TOKEN`). |
| `APIFY_INDEED_TOKEN` / `INDEED_API_KEY` | – | Token for Indeed only. |
| `APIFY_MONSTER_TOKEN` / `MONSTER_API_KEY` | – | Token for Monster only. |
| `DEFAULT_COUNTRY` | `in` | Two-letter country code pre-selected in the UI and used when a request omits it. |
| `APIFY_TIMEOUT_SECS` | `290` | Per-actor timeout. The Apify synchronous endpoint caps at about 300 s. |
| `DB_PATH` | `backend/data/jobmatch.db` | Location of the SQLite file. |

Notes:

* A token may be given as a bare token **or** as a full URL containing `?token=…`; the token part is extracted automatically.
* Values containing `your_` (the placeholder in `.env.example`) are ignored.
* Tokens are sent to Apify in an `Authorization: Bearer` header, never in the URL, and are scrubbed from every error message before it is logged or returned.

---

## 6. How it works

### 6.1 Fetching (`backend/sources/`)

`providers.py` defines one `Provider` per board: key, label, Apify actor id, an `build_input()` function that turns the generic search parameters into that actor's input JSON, and an optional set of supported countries.

`base.run_actor()` POSTs the input to
`https://api.apify.com/v2/acts/{actorId}/run-sync-get-dataset-items` and maps HTTP errors to readable hints (401 token rejected, 402 out of credits, 403 not allowed, 404 actor missing, 408 time limit).

### 6.2 Normalisation (`base.normalize_item`)

Each actor returns different field names. `normalize_item` tolerates many aliases (for example `companyName` / `company` / `employer.name`) and produces one shape for the UI:

```json
{
  "id": "live-3f9a1c2b7d10",
  "title": "GenAI Engineer",
  "company": "Globex",
  "location": "Pune, MH, India",
  "experience": "3–5 yrs",
  "workType": "Hybrid",
  "salary": "INR 1,200,000 - INR 2,000,000 / YEAR",
  "description": "…first 900 characters…",
  "requiredSkills": ["Python", "LangChain", "RAG"],
  "matchingSkills": [], "missingSkills": ["Python", "LangChain", "RAG"],
  "matchScore": 0, "recommendationReason": "",
  "source": "LinkedIn",
  "postedDate": "2d ago",
  "url": "https://…",
  "live": true
}
```

* `id` is a SHA-1 of the posting URL (or source+title+company+location when no URL), so re-fetching the same posting updates it instead of duplicating it.
* HTML descriptions are stripped to text.
* `matchScore`, `matchingSkills`, `missingSkills` and `recommendationReason` are filled in later by the scorer.

### 6.3 Extraction (`backend/skills.py`)

* **Skills** – a vocabulary of ~100 technologies, each with aliases (`k8s` → Kubernetes, `sklearn` → Scikit-learn, …). Matching is case-insensitive on word boundaries. Ambiguous aliases that are also ordinary English words (`rest`, `react`, `excel`, `spark`, `rag`, `ml`, `js`, `torch`, `rust`) only match with their normal capitalisation (`REST`, `React`, `Excel`…), and `Go` only matches next to words such as *developer*, *engineer* or *backend*. Skills are returned in order of first mention (earliest = most important), capped at 14.
* **Experience** – "N years / N–M years / N+ years" counts only when the words *experience*, *expertise* or *background* are nearby (so "in business for 15 years" is ignored). The lowest stated minimum wins and is bucketed into `0–2 yrs`, `3–5 yrs`, `6–9 yrs`, `10+ yrs`, otherwise `Not specified`. Junior / intern / fresher titles map to `0–2 yrs`.
* **Work type** – `Remote`, `Hybrid`, `On-site` or `Not specified` from the actor's flag/fields and keywords in the text (hybrid takes precedence over remote).

---

## 7. Scoring algorithm

`matching.score_job(job, candidate_skills, role, location, feedback)` returns the job plus `matchScore` (0–100), `matchingSkills`, `missingSkills` and a human-readable `recommendationReason`.

```text
coverage      = matching skills / min(required skills, 8)       (capped at 1.0)
skill points  = 80 × coverage
title fit     = +12 if the candidate's role and the job title share a meaningful word
location fit  = +8  if the candidate's city appears in the job location
                +4  if the job is Remote and the city does not match
feedback      = +3 for 👍 ("up"/"relevant"),  −8 for 👎 ("down"/"irrelevant")

matchScore    = clamp( round(skill points + title fit + location fit + feedback), 0, 100 )
```

Details:

* Skills are compared after normalisation through the alias table, so `"OpenAI"`, `"openai api"` and `"Open AI API"` are equal, as are `"MCP Tools"` and `"MCP"`.
* The denominator is capped at 8 so a posting that lists 14 skills does not punish a candidate who covers the first 8.
* Title fit ignores seniority and generic words (*senior, lead, engineer, developer, manager, analyst, …*) so every "Engineer" role does not receive the bonus, but keeps two-letter terms such as **AI** and **ML**.
* A posting with no detected skills is scored on title and location only.

Worked example — job requires `Python, RAG, Docker`, candidate knows `Python, RAG`, role matches, city matches:
`80 × (2/3) = 53.3` + 12 + 8 = 73.3 → **73**.

---

## 8. Data model

SQLite file, three tables, created automatically (`backend/db.py`). The connection uses WAL mode so the background fetch threads can write while the UI reads.

| Table | Columns | Notes |
|---|---|---|
| `jobs` | `id` (PK), `source`, `payload` (JSON), `fetched_at` | Live jobs only. Upsert on `id`. |
| `applications` | `id` (PK), `job_id`, `payload` (JSON), `created_at` | One application per job. |
| `kv` | `key` (PK), `value` (JSON) | Keys: `profile`, `feedback` (`{job_id: "up"|"down"|…}`). |

Sample jobs and the default profile are in `backend/seed.py` and are **not** written to the database.

Reset everything by stopping the server and deleting `backend/data/jobmatch.db`; clear only the live jobs with `DELETE /api/jobs`.

---

## 9. REST API reference

Base URL `http://127.0.0.1:8000`. All bodies are JSON. Errors use FastAPI's `{"detail": "…"}` shape.

### System

| Method & path | Description |
|---|---|
| `GET /` | Service info and links. |
| `GET /api/health` | `{status, service, mode: "sample"\|"live", jobs_loaded, sources_configured}` |
| `GET /api/sources` | Boards and whether a token exists for each (never returns tokens). |

### Jobs

#### `GET /api/jobs`

Stored jobs (live if any, otherwise sample), scored for the saved profile and sorted by `matchScore` descending.

Query parameters: `search` (title / company / skill substring), `work_type`, `experience`, `source` — each ignored when `All`.

```json
{"status": "success", "mode": "live", "total": 42, "jobs": [ { "...job fields...": "", "matchScore": 87 } ]}
```

#### `POST /api/jobs/refresh`

Start a live fetch.

| Field | Type | Default | Notes |
|---|---|---|---|
| `query` | string, 2–120 chars | required | Job title / keywords |
| `location` | string | none | City or region |
| `country` | 2-letter code | `DEFAULT_COUNTRY` | e.g. `in`, `us`, `gb` |
| `work_type` | `Remote`\|`Hybrid`\|`On-site` | none | Used by LinkedIn |
| `sources` | list of `linkedin`, `indeed`, `monster` | all | |
| `limit` | int 1–100 | 25 | Results per board |

Response:

```json
{"status": "started", "run_id": "a1b2c3d4e5", "sources": ["linkedin", "indeed"],
 "skipped_no_token": [], "skipped_unsupported_country": ["monster"]}
```

Errors (`400`): no valid source; every requested board is unsupported for the country; no token configured.

#### `GET /api/jobs/refresh/{run_id}`

```json
{"run_id": "a1b2c3d4e5", "status": "running|done|error", "total": 37, "elapsed": 64,
 "sources": {"linkedin": {"status": "done", "count": 25},
             "indeed":   {"status": "error", "count": 0, "error": "Apify account is out of credits…"}}}
```

A run is `done` if at least one board returned jobs (or at least one board did not fail), `error` if every board failed. `404` for an unknown id (the last 20 runs are kept in memory).

#### `DELETE /api/jobs`

Removes all stored live jobs; the app falls back to sample data.

#### `GET /api/sources/{source}/sample`

Developer aid: runs one actor with `limit=3` and returns both the **raw** and **normalised** records so you can confirm the field mapping. Costs Apify credits.

### Recommendations

#### `POST /api/recommend`

```json
{"skills": ["Python", "RAG"], "role": "GenAI Engineer", "location": "Pune, MH", "work_type": "Remote"}
```

All fields optional; empty `skills` falls back to the saved profile. Returns `recommendations` sorted by `matchScore`.

### Profile

| Method & path | Body | Description |
|---|---|---|
| `GET /api/profile` | – | Current profile (default from `seed.py`). |
| `PUT /api/profile` | `{name, title, location, email, phone, skills[]}` | Replace the profile. All fields required. |

### Applications

| Method & path | Body | Description |
|---|---|---|
| `GET /api/applications` | – | All applications, newest first. |
| `POST /api/applications` | `{job_id}` | Create (idempotent per job). `404` if the job does not exist. |
| `PATCH /api/applications/{id}` | `{status}` | Status ∈ `Applied, Reviewing, Interviewing, Offered, Rejected`. |

### Feedback

`POST /api/feedback` with `{job_id, feedback}` where `feedback` ∈ `up, down, relevant, irrelevant`. Feedback changes that job's score (+3 / −8) on every later request.

### MCP simulation

| Method & path | Description |
|---|---|
| `GET /api/mcp/tools` | The eight simulated tools and their status. |
| `POST /api/mcp/pipeline` | Runs the real scoring pipeline and reports a message per step. |

---

## 10. Frontend guide

`frontend/index.html` is served by FastAPI at `/app/` and can also be opened directly from disk (it then talks to `http://127.0.0.1:8000`).

| Tab | Purpose |
|---|---|
| **Dashboard** | Welcome banner, headline statistics and the top-match spotlight. |
| **Job Search Engine** | Live-fetch panel, text search, work-type and experience filters, job cards, job detail modal, save/apply. |
| **AI Feed** | Rule-based recommendations ranked by score, with 👍/👎 feedback. |
| **Tracker** | Application tracker with status changes. |
| **Candidate Profile** | Edit name, title, location, contact and skills (adding/removing skills re-ranks jobs). |
| **MCP Tools** | Simulated tool inspector and pipeline runner. |
| **Settings** | Match-cutoff threshold and display options (stored in the browser). |

Behaviours worth knowing:

* If the backend is unreachable the UI falls back to the built-in sample data and shows **OFFLINE**.
* Profile and applications are mirrored to `localStorage` and to the backend. On load the UI merges applications stored in the backend so they survive a cleared browser.
* **Monster** is greyed out unless the country is *United States* (see §11).
* Everything coming from scraped job text or from the server is HTML-escaped before it is placed in the page, and only `http(s)` links are allowed.

---

## 11. Job-board sources (Apify actors)

| Board | Actor | Apify id | Countries |
|---|---|---|---|
| LinkedIn | `curious_coder/linkedin-jobs-scraper` | `hKByXkMQaC5Qt9UMN` | all (location text) |
| Indeed | `valig/indeed-jobs-scraper` | `TrtlecxAsNRbKl1na` | ISO code in `country` |
| Monster | `blackfalcondata/monster-scraper` | `9d4mn9SVh5WvvtLii` | **United States only** |

Inputs built by `providers.py` (checked against each actor's published input schema):

* **LinkedIn** – a `urls` entry with a LinkedIn search URL (`keywords`, `location`, and `f_WT` = 1 on-site / 2 remote / 3 hybrid), plus `limitPerSource`.
* **Indeed** – `country`, `title`, `location`, `limit`, `datePosted: "7"` (last 7 days).
* **Monster** – `query`, `location`, `country: "US"`, `maxResults`.

Costs and timing: each fetch consumes Apify credits and takes about 30–120 s per board; keep *Results per board* modest.

If an actor changes its output format, run `GET /api/sources/<board>/sample`, compare `raw` with `normalized`, and extend the alias lists in `normalize_item` (`backend/sources/base.py`).

---

## 12. Testing

```bash
pip install -r requirements.txt -r requirements-dev.txt
pytest
```

37 tests, no network and no Apify credits needed:

| File | Covers |
|---|---|
| `tests/test_skills.py` | skill extraction (aliases, ambiguous words, Go), experience buckets, work type |
| `tests/test_normalize_and_matching.py` | Indeed/LinkedIn record mapping, salary/date helpers, score formula, feedback, clamping |
| `tests/test_api.py` | every endpoint, live refresh with a faked actor, partial/total board failure, token scrubbing, Monster country rule, application lifecycle, frontend assets |

`tests/conftest.py` points `DB_PATH` at a temporary file and sets dummy tokens **before** importing the app, so tests never touch your real database or `.env`.

---

## 13. Security notes

* **Rotate any token that has been shared.** Earlier revisions of this project carried tokens in `api.txt`. Create a new token at <https://console.apify.com/settings/integrations> and keep it only in `.env`. `.env` and `backend/data/` are git-ignored; do not zip or upload them.
* The API has no authentication and CORS allows all origins. It is intended for local use. Put authentication in front of it (and restrict CORS) before exposing it.
* Scraped text is untrusted. It is escaped in the UI, and links are limited to `http(s)`.
* Scraping job sites can conflict with their terms of service; make sure your use complies.

---

## 14. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| UI shows **OFFLINE** | Backend not running. Start it with `uvicorn backend.main:app --reload`. |
| "No Apify token configured" | `.env` missing or still has the placeholder; set `APIFY_TOKEN` and **restart** the server. |
| Board shows `failed: Token rejected by Apify` | Wrong/rotated token. |
| `failed: Apify account is out of credits…` | Add credits or use a plan that can run the actor (HTTP 402). |
| `failed: Actor run exceeded the time limit` | Reduce *Results per board* (HTTP 408). |
| Monster checkbox is disabled | The Monster actor only covers the US; choose United States. |
| Salary/location empty on live jobs | Call `/api/sources/<board>/sample` and extend the aliases in `normalize_item`. |
| Old jobs from a previous search still appear | Live jobs accumulate across fetches. Use `DELETE /api/jobs` to clear them. |
| `TypeError: unsupported operand type(s) for |` on startup | Python older than 3.10. Upgrade Python. |
| Icons missing | Make sure `frontend/vendor/lucide.min.js` exists (it ships with the project). |

---

## 15. Known limitations and roadmap

* Scoring is **rule-based**, not a trained or generative model. Natural next steps: embeddings for semantic matching, an LLM for résumé parsing and match explanations, and a real MCP server behind the MCP tab (the *Resume Parser* tool is a placeholder).
* `postedDate` is converted to text such as "2d ago" at fetch time and stored that way, so it goes stale for old rows.
* Live jobs accumulate across searches until cleared; there is no automatic expiry.
* There is a single profile and a single user; no login.
* The frontend is one large file with inline script; splitting it into modules would make it easier to maintain.

---

## 16. Changelog — fixes in this revision

**Job sources**

* **Monster** only supports the US, but the app sent the default country (`IN`). The input now always uses `US`, passes the location, and the API skips Monster (with `skipped_unsupported_country`) for other countries; the UI disables the checkbox.
* **Indeed** records now map correctly: `datePublished` (posted date), `baseSalary.currencyCode` / `unitOfWork` (currency and period), and structured `location` objects (city, state, country).
* A non-JSON reply from Apify now produces a clear `SourceError` instead of an unhandled exception.

**Parsing and scoring**

* `golang` was never detected (the `Go` entry was skipped entirely); fixed.
* Everyday words no longer count as skills (*rest*, *react*, *excel*, *spark*, …) unless capitalised as the technology.
* Experience buckets overlapped (3–4 years and 5 years landed in different, overlapping buckets); they are now `0–2 / 3–5 / 6–9 / 10+`. Company boilerplate such as "15 years in business" is ignored.
* Two-letter terms such as **AI** were dropped from the title-fit check, while generic words like *engineer* gave every engineering job the bonus; both fixed.
* `GET /api/jobs` now returns jobs sorted by score.

**Backend robustness**

* SQLite uses WAL mode, a longer busy timeout and rolls back on errors (three fetch threads write concurrently).
* A refresh run can no longer stay stuck in `running`, and cannot crash if it was evicted from the run table.
* `country` must be a two-letter code.

**Frontend**

* Icons are served from `frontend/vendor/` instead of a third-party CDN, so the UI works offline.
* Skill names are HTML-escaped and removed through a `data-` attribute (a skill containing a quote or `<` broke the tag or injected markup).
* Applications stored in the backend are loaded and merged on start-up (previously only `localStorage` was read).

**Project hygiene**

* Added a 37-test pytest suite, `requirements-dev.txt` and `pytest.ini`.
* Added this documentation and refreshed the README.

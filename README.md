---
title: KhojAPI
emoji: 🔍
colorFrom: green
colorTo: green
sdk: docker
pinned: false
---
# khojAPI
A local business intelligence agent for Indian markets. A multi-agent system that takes a city, locality, and business category, crawls Google Places, extracts structured insights using LLaMA 3.3, runs VADER sentiment analysis on real customer reviews, and returns competition density, gap signals, and complaint patterns via a REST API.

## 1. Problem Statement
Open track. A founder looking to open a business in a specific locality has no way to find out all three in one go: market saturation, what customers are complaining about with existing players, and which price band is underserved. Manual research on Google would take hours. khojAPI solves this in under 60 seconds.


## 2. Tech Stack
- **Backend:** FastAPI, Python 3.11, SQLite, SQLAlchemy
- **Agent layer:** Groq API, LLaMA 3.3 70B, Pydantic v2
- **Crawling:** Google Places Text Search API, Google Places Details API
- **Analytics:** VADER sentiment, pandas
- **Frontend:** HTML, CSS, vanilla JS
- **Deployment:** Docker, HuggingFace Spaces

## 3. Backend Architecture
Three agents with single responsibility:

- **Crawler agent** — fetches businesses from Google Places Text Search, then calls Place Details API per business for reviews and phone number
- **Extractor agent** — sends raw business data to LLaMA 3.3 via Groq, returns structured JSON with price range, complaints extracted from reviews, and tags
- **Analytics agent** — runs VADER sentiment scoring on complaint text, computes competition density, identifies underserved price band

Orchestrator chains them: crawl → extract → analyze → persist to SQLite. Async job queue so the API returns immediately with a job_id while processing happens in the background.

**Why these decisions:**
- Async over sync — Place Details calls for 20 businesses take 15-20 seconds, synchronous would timeout
- SQLite over Postgres — zero infrastructure overhead for demo, swap path documented
- VADER over LLM for sentiment — free, local, instant, strong accuracy on short review text
- Three agents over one — each is testable in isolation, single responsibility principle

## 4. Implementation Approach & Workflow
1. User submits city, locality, and category via the frontend or API
2. `POST /analyze` creates a job in SQLite and returns a `job_id` immediately
3. Background task fires the pipeline:
   - Crawler calls Google Places Text Search to find businesses in that locality
   - For each business, a second Place Details call fetches real customer reviews (Google returns up to 3 reviews per business, sorted by relevance — most helpful first) and phone number
   - Extractor sends each business's raw data to LLaMA 3.3 via Groq — LLM returns structured JSON with price range, complaints extracted from reviews, and tags
   - Analytics runs VADER sentiment scoring on complaint text, computes competition density, identifies which price band is underserved
   - All businesses and analytics saved to SQLite, job marked complete
4. Client polls `GET /status/{job_id}` every 2 seconds
5. Once complete, `GET /report/{job_id}` returns the full structured report
6. Frontend renders business cards, sentiment labels, metric boxes, and market opportunity banner

## 5. Features & Functionalities
- **Micro-locality search** — search by neighborhood, not just city
- **Real review extraction** — up to 3 reviews per business from Google Places, sorted by relevance
- **LLM-powered complaint extraction** — identifies what customers actually complain about
- **Sentiment scoring per business:**
  - Above 0.2 → Positive
  - 0 to 0.2 → Mixed
  - Below 0 → Negative
- **Competition density scoring** — low/medium/high based on business count in locality
- **Underserved price band detection** — identifies which tier (budget/mid-range/premium) is missing
- **Async job queue** — non-blocking, pollable via status endpoint

## 6. API Endpoints
- `POST /analyze` — body: city, locality, category — returns job_id
- `GET /status/{job_id}` — returns pending/processing/complete
- `GET /report/{job_id}` — returns full structured report

## APIs / Models / Tools Used
- **Google Places Text Search API** — find businesses by locality and category
- **Google Places Details API** — get reviews and phone per business
- **Groq API with LLaMA 3.3 70B** — structured extraction from raw data
- **VADER** — local sentiment scoring, no API cost

## 7. Installation Steps

**With Docker:**
```bash
git clone https://github.com/sofian229/khojAPI
cd khojAPI
cp .env.example .env  # add your API keys
docker-compose up --build
```

**Without Docker:**
```bash
git clone https://github.com/sofian229/khojAPI
cd khojAPI
python -m venv .venv
.venv\Scripts\activate  # Windows
pip install -r requirements.txt
cp .env.example .env  # add your API keys
uvicorn main:app --reload
```

## 8.  Screenshots
<img width="1915" height="982" alt="image" src="C:\Users\abdul\Downloads\khojAPI-main\khojAPI-main\screenshots\Screenshot 2026-11-10 234331.png" />
<img width="1912" height="986" alt="image" src="C:\Users\abdul\Downloads\khojAPI-main\khojAPI-main\screenshots\Screenshot 2026-11-10 234510.png" />


## 9. Known Limitations
- Google Places returns a maximum of 3 reviews per business on the free tier, sorted by relevance. Businesses with no recent reviews will show empty complaints.
- SQLite is used for this demo. A production version would use Postgres.
- Complaint extraction depends on review availability — businesses with no Google reviews will show no complaints.

## 10. Environment Variables Required
Create a `.env` file based on `.env.example`:

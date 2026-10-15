# AI News Summarizer & Analyzer (Django + NLP + LLM)

Paste a news URL (or the article text) and get a summary, key points, sentiment, category, keywords,
named entities, a bias/framing note and a list of claims worth verifying. A dashboard shows trends
across everything you've analyzed.

## How it works

```
URL / pasted text
      │
fetcher.py ── SSRF-safe download (blocks private IPs, checks every redirect, 2 MB cap) + BeautifulSoup cleaning
      │
nlp.py ────── offline baseline: VADER sentiment, TF keywords/bigrams, entities (spaCy if installed),
      │       frequency-based extractive summary, keyword category classifier
      │
llm.py ────── Claude returns strict JSON (summary, key points, bias notes, entities, claims to verify);
      │       output is validated/clamped; article text is treated as data, not instructions
      │
pipeline.py ─ LLM overlays the NLP baseline; any LLM failure silently falls back to NLP only
      │
SQLite (Article + Analysis) ──► detail page, dashboard (Chart.js), JSON API, RSS ingest command
```

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # add ANTHROPIC_API_KEY (optional) and API_TOKEN (optional)
python manage.py migrate
python manage.py createsuperuser   # optional, for /admin
python manage.py runserver
```

Open http://127.0.0.1:8000. Without `ANTHROPIC_API_KEY` the app still works using the NLP-only engine.

Better entities (optional): `pip install spacy && python -m spacy download en_core_web_sm`

## Usage

- **Web UI**: `/` analyze, `/article/<id>/` results, `/dashboard/` trends.
- **RSS ingest**: `python manage.py ingest_feeds --limit 5` (or `--feed URL`, `--no-llm`).
- **JSON API** (requires `API_TOKEN` in `.env`; disabled otherwise):

```bash
curl -X POST http://127.0.0.1:8000/api/analyze/ \
  -H "Authorization: Bearer $API_TOKEN" -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/some-news-story"}'
```

- **Tests**: `python manage.py test`

## Notes before deploying

- Set `DJANGO_DEBUG=0`, a real `DJANGO_SECRET_KEY`, and `DJANGO_ALLOWED_HOSTS`.
- The analyze form is public and each submit can call the paid LLM: add login, throttling or a daily cap first.
- Analysis is only as good as the extracted text; paywalled / JS-rendered pages fall back to a "paste the text" message.
- The "bias" and "claims to verify" fields are model judgments, not fact-checks.

## Ideas to extend

Celery for background analysis, topic clustering with embeddings, multi-article comparison of the same story,
translation for non-English news, Postgres + full-text search.

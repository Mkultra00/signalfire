# SignalFire

**Creator Economy Hack, powered by You.com** (NYC, 2026-11-22). Track: **Trend Radar**.

> SignalFire: type your niche, get the 3 trends and memes rising on today's web with You.com sources, what to make first (focus, hook, timing, format), a live trend tree of the scan, and a brief that refuses to write itself without citations.

## The challenge

Mid-tier independent creators and solo social managers plan 1–3 days ahead and burn an hour a day hunting trends they can't verify. SignalFire takes a niche, runs live You.com searches (trend, news, meme), and ranks the 3 freshest, most corroborated topics with a transparent heuristic score and visible source links. It then tells you what to make first: the focus topic, hook options, a timing window, and a content format, all derived from the cited sources. A Gource-style tree replays how the scan found each signal. One click turns a topic into a creator brief. If there are no usable sources, it refuses. No invented URLs, no fake stats, no "this will go viral."

## How You.com is used

- **Search API** is the engine. Each scan fans out 3 fresh-web queries in parallel (`{niche} trend`, `{niche} news`, `{niche} meme`; freshness = past week, country US). Every topic, score input, citation, recommendation, and trend-tree event comes from those results.
- **Contents API** powers the brief. It fetches the cited pages (up to 3 per brief) so the "why now" lines are quoted from the live source rather than generated.

No You.com result means no topic. No usable citation means no recommendation and no brief. There is no LLM, no mock mode and no demo data in the product: with no key the app returns 503.

## Run

Needs Python 3.10+.

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
# Set the environment variable YOU_API_KEY in your own shell (never in a file, never committed).
uvicorn signalfire.app:app --host localhost --port 8791
```

Open `http://localhost:8791`. The app binds to loopback only. Ports 8080, 8766 and 8768 are not used.

## Test

```bash
python -m pytest -q
```

All You.com HTTP is mocked with `respx` (fixtures use reserved `example.*` domains). The live smoke test is skipped unless `RUN_LIVE=1` and `YOU_API_KEY` are both set; it costs about 3 searches.

## API

| Route | What it does |
|---|---|
| `GET /` | The single page (vanilla JS, inline, no CDN, strict CSP) |
| `POST /api/scan` `{"niche": "..."}` | 3 topics + HEURISTIC scores + citations, recommendations, Gource log. 400 bad niche, 503 no key, 502 upstream down, 422 not enough fresh sources (never padded) |
| `POST /api/brief` `{"topic_id": "..."}` | Cited brief with disclaimer. 404 unknown topic, **409 refusal with zero citations**, 503 no key |
| `POST /api/gource` | The last successful scan's Gource custom log (`unix\|role\|A/M/D\|/path`), 404 before any scan |

**Score:** a hand-set **HEURISTIC**: `0.40·recency + 0.30·corroboration + 0.20·cross-query fan-out + 0.10·news`, shown 0–10. Not trained, not calibrated, not a prediction.

## Honest limits

- **The heuristic is unvalidated.** Weights are hand-set and scores are relative within one scan. "Predictive" means forward-looking rules over early signals; we make no accuracy claim.
- **Recommendations are rules, not judgment.** Focus is the top cited topic, hooks are templates plus one verbatim snippet, timing is a recency bucket, format is a 4-way rule.
- **"Meme" means query membership.** A result is a meme signal because You.com returned it for `{niche} meme`, not because we classified it. A meme can rank low.
- **A "topic" is a headline.** Topics are real article titles grouped by a crude shared-token rule; near-duplicates are possible when distinctness is relaxed to fill 3.
- **The tree is synthetic.** Paths are slugs of niche/platform/title; timestamps are source `page_age` clamped to 7 days. It replays one scan, it is not a live crawl and not git history. Slugging drops dotted tokens such as "2.0".
- **Recency depends on `page_age`.** Undated results score 0 on recency, get "Timing unclear", and appear at scan time in the tree.
- **Verbatim snippets can contain any words.** Quotes are attributed source text with `[n]`, never SignalFire's claim.
- **A live refusal can't be forced.** Whether a scan yields a zero-citation topic depends on the web.
- **Session memory only.** A restart loses topics and the last log. Single process, localhost-bound, no rate limiting.
- **You.com details are partly unverified.** See `BUILD_LOG.md`: no live call has been made from this build yet. Current docs mix news into `web` results, so the news signal may rarely fire live.
- **US/English only.** `country=US`, English stopwords.
- **Cost.** A scan is 3 searches, a brief is 1 Contents call. Never put scans in a loop.
- **Latency.** The ~15 s target assumes You.com answers within 8 s per call; slower shows "Still searching", a timeout is a visible 502.

Out of scope: posting or scheduling, TikTok/YouTube APIs or scraping, LLM writing, training or evals, React or any build step, databases, auth, deployment, the Gource binary.

## Layout

```
requirements.txt
signalfire/you_client.py     You.com Search + Contents, key from env only, generic errors
signalfire/ranking.py        merge, roles, HEURISTIC score, blocklist, pick exactly 3
signalfire/recs.py           focus / hooks / timing / content rules over cited topics
signalfire/gource.py         Gource custom-log builder with safe slugs
signalfire/brief.py          cite-or-refuse brief
signalfire/app.py            FastAPI routes, error mapping, in-memory state
signalfire/static/index.html single page: cards, recs, SVG trend tree, brief
tests/                       pytest + respx, fixtures, opt-in live smoke test
docs/TDD-signalfire.md       the spec
```

## Team

- Frank Yu — [Mkultra00](https://github.com/Mkultra00)
- Forrest Pan — [panforrest](https://github.com/panforrest)

## License

MIT. See `LICENSE`.

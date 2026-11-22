# SignalFire: Technical Design Doc (TDD), v2

Creator Economy Hack, powered by You.com, NYC, Sun 2026-11-22. Hacking and Devpost close at **16:00 EST**.
Sources: `inputs/REQUIREMENTS.md`, `inputs/PRODUCT.md`, `inputs/event-packet.md`, `inputs/research.md` (background only; **PRODUCT.md wins on any conflict**). Supersedes `docs/TDD-signalfire-v1.md`. v1 contracts that still fit are carried over unchanged. New in v2: meme signal, recommendations, Gource-format trend log, `/api/gource`, the in-page tree UI, and timing/content tips in the brief.

This doc is for the implementing pass. Write every test **before** the code it covers. Where this doc gives a number, a string, a field name, or a path, use it exactly. Labels:

- **[CONTRACT]**: behavior that tests pin. Change it only by changing this doc.
- **[DECISION]**: a choice made so the implementer doesn't have to guess. Reverse it only by editing this doc.
- **[LIVE]**: behavior only real You.com traffic can show. Tests mock it.
- **[GAP]**: a known limit, stated honestly. Don't hide it, and don't fix it today.
- **[VERIFY]**: check against the live You.com docs or a real response during implementation. If reality differs, update the fixture and the parser **together**, in one place, and note the change here.

---

## 1. Goal, track, one-liner

**Goal (one sentence):** A creator types a niche and within about 15 s gets exactly 3 rising trends/memes from today's web, each with visible You.com sources and a documented HEURISTIC score. They also get recommendations for what to make first (focus, hooks, timing, format), a Gource-format trend tree of the scan, and a brief that **refuses** to write itself without citations.

**Devpost track:** **Trend Radar** (one track only). The cite-or-refuse brief is how Uniqueness and Research-Desk-style honesty show up. We do **not** enter Research Desk, and we don't submit paper-only.

**One-line description (paste into Devpost):**
> SignalFire: type your niche, get the 3 trends and memes rising on today's web with You.com sources, what to make first (focus, hook, timing, format), a live trend tree of the scan, and a brief that refuses to write itself without citations.

**Problem / creators / what it does (paste into Devpost):**
> Mid-tier independent creators and solo social managers plan 1–3 days ahead and burn an hour a day hunting trends they can't verify. SignalFire takes a niche, runs live You.com searches (trend, news, meme), and ranks the 3 freshest, most corroborated topics with a transparent heuristic score and visible source links. It then tells you what to make first: the focus topic, hook options, a timing window, and a content format, all derived from the cited sources. A Gource-style tree replays how the scan found each signal. One click turns a topic into a creator brief. If there are no usable sources, it refuses. No invented URLs, no fake stats, no "this will go viral."

**How You.com is used (paste into Devpost):**
> You.com **Search API** is the engine. Each scan fans out 3 fresh-web queries (trend, news, meme; freshness = past week). Every topic, score input, citation, recommendation, and trend-tree event comes from those results. You.com **Contents API** powers the brief: it fetches the cited pages so the "why now" lines are quoted from the live source rather than generated. No You.com result means no topic. No usable citation means no recommendation and no brief.

**[DECISION] "Predictive" wording.** In the pitch, "predictive" means *forward-looking recommendations from a HEURISTIC over early public signals*. It does not mean a trained model. Copy in the UI and README says "early signal" and "HEURISTIC", never "prediction model" or "will go viral."

---

## 2. Architecture

A single FastAPI process on `127.0.0.1:8791` serves one static `index.html` (vanilla JS, inline, no build step, no CDN) and three JSON endpoints. `POST /api/scan` fans out 3 You.com Search calls in parallel, then merges and ranks the results with a pure-Python HEURISTIC. From the top 3 topics it derives rule-based recommendations and builds a Gource-format custom log, and it keeps topics and the log in process memory. `POST /api/brief` refuses with 409 when a topic has zero citations; otherwise it fetches cited pages via You.com Contents and fills a template that quotes only that evidence. `POST /api/gource` returns the last scan's log, which the page plays as an SVG tree. There is no LLM, database, Redis, React, or Gource binary.

**[DECISION] Port `8791`.** Never 8080, 8766, or 8768. Before first start, run `lsof -nP -iTCP:8791 -sTCP:LISTEN`. If it's taken, use 8792 and update the README. **[VERIFY]**

**[DECISION] No LLM.** Briefs and recommendations are deterministic templates plus text copied verbatim from You.com results. This guarantees "no invented sources, no invented stats" by construction, and it isn't a frozen-model wrapper, because without You.com there is no output.

**[DECISION] No mock or demo mode in product code.** The app uses live You.com or returns 503. Fakes live only in `tests/`.

**[DECISION] Injected clock.** `create_app(clock=...)` takes a `Callable[[], datetime]` that defaults to `lambda: datetime.now(timezone.utc)`. Tests pass a fixed `NOW` so golden scores, timing, and log timestamps are deterministic.

### Files expected after the implement pass (exact paths)

```
requirements.txt                 # fastapi, uvicorn[standard], httpx, pytest, respx
.gitignore                       # .env, .env.*, __pycache__/, .pytest_cache/, .venv/, *.log
README.md                        # run, test, Devpost text from §1, "set env var YOU_API_KEY" (no value)
signalfire/__init__.py           # empty
signalfire/you_client.py         # You.com HTTP: require_key(), search(), contents(); errors
signalfire/ranking.py            # Hit→Candidate merge, roles, HEURISTIC score, eligible(), pick(), rank(), blocklist
signalfire/recs.py               # format_for(), hooks_for(), timing_for(), build()  (focus/hooks/timing/content)
signalfire/gource.py             # safe_slug(), platform(), build_log(), LINE_RE
signalfire/brief.py              # DISCLAIMER, refusal(), build_brief()
signalfire/app.py                # create_app(clock), routes, error mapping, in-memory state; `app = create_app()`
signalfire/static/index.html     # single page: input, 3 cards, recs panel, brief panel, SVG Gource tree
tests/__init__.py                # empty (so `from tests.conftest import ...` resolves)
tests/conftest.py                # env fixtures, respx You.com mock, TestClient w/ fixed clock, loader
tests/fixtures/search_ok.json    # §5 fixture spec
tests/fixtures/search_meme.json  # §5 fixture spec
tests/fixtures/contents_ok.json  # §5 fixture spec
tests/test_you_client.py
tests/test_ranking.py
tests/test_recs.py
tests/test_gource.py
tests/test_brief.py
tests/test_api.py
tests/test_ui.py
tests/test_live_smoke.py         # skipped unless RUN_LIVE=1 and YOU_API_KEY set
```

No other files. No `pyproject.toml`, `package.json`, `.env`, `.env.example`, JS bundles, or `.gource` binaries.

Run: `uvicorn signalfire.app:app --host 127.0.0.1 --port 8791`
Test: `python -m pytest -q`

Module dependency (one direction only): `you_client` ← `ranking` ← `recs` ← `gource`, `brief` ← `app`. `you_client.contents` imports `ranking.normalize_url` lazily inside the function to avoid a cycle.

---

## 3. You.com engine

| Endpoint | Called by | Powers | Calls per action |
|---|---|---|---|
| **Search**: `GET https://ydc-index.io/v1/search` **[VERIFY]** | `you_client.search()` | Topic candidates, titles, snippets, freshness (`page_age`), news-vs-web flag, meme-query membership, cross-query corroboration. These feed **all** ranking inputs, citations, recommendation evidence (hooks quote snippets, timing uses freshness), and every Gource event. | 3 per scan, in parallel |
| **Contents**: `POST https://ydc-index.io/v1/contents` **[VERIFY]** | `you_client.contents()` | Page text for a topic's cited URLs. The brief's "why now" quotes are extracted from it. | 1 per brief (batched, ≤ 3 URLs) |

**[DECISION]** Research and Answer are not used in the MVP. Their shape, cost, and latency are unverified, and Search + Contents covers every feature. Contents is not called during the scan, to keep the scan under 15 s.

**Auth [CONTRACT]:** The key is read only from env var `YOU_API_KEY`, at call time, through `require_key()`. It's sent as header `X-API-Key` **[VERIFY header name]**. It is never logged, echoed in exceptions or responses, or written to any file. There is no default and no fallback. This doc, the README, and code samples name the variable but **never** contain a value.

**Search request [VERIFY param names]:** `query=<q>`, `count=10`, `freshness=week`, `country=US`.

**Search response parsing [VERIFY shape]:** Expect `{"results": {"web": [...], "news": [...]}}`. Each item may have `url`, `title`, `description`, `snippets: [str]`, `page_age` (ISO-8601). Parse tolerantly:
- A missing `results`, `web`, or `news` counts as `[]`.
- `snippet = snippets[0]` if non-empty, else `description`, else `""`. Strip it and cut it to 240 chars.
- A missing or unparseable `page_age` becomes `None`.
- Items with no `url` or no `title` are dropped at parse time.
- Output order: all `web` items in order, then all `news` items in order.

**Contents request [VERIFY]:** JSON body `{"urls": [...], "format": "markdown"}`.
**Contents response [VERIFY]:** A list of objects. Page text is the first non-empty of `markdown`, `text`, `html`, `content`. The result is a `dict[normalize_url(url), text]`, and empty-text items are omitted.

**Timeouts [DECISION]:** `httpx.Timeout(8.0)` per call. The 3 scan queries run in a `ThreadPoolExecutor(max_workers=3)`, so the worst case is about 8 s of wall time.

### Missing and failed data [CONTRACT]

| Situation | Behavior |
|---|---|
| `YOU_API_KEY` unset or blank | `EngineUnavailable` is raised **before any HTTP call** → 503 `{"error":"engine_unavailable","message":"Live search is not configured."}` |
| 1–2 of 3 scan queries fail (timeout, 4xx, 5xx, bad JSON) | The failed query's slot becomes `[]` (its position is kept, so the meme index stays aligned). Ranking continues. |
| All 3 queries fail | 502 `{"error":"upstream_error","message":"Live search failed. Try again."}` |
| < 3 eligible candidates | 422 `{"error":"insufficient_signal","message":"Not enough fresh sources for this niche. Try a broader niche.","found":N}`. **Never pad.** |
| Result with no snippet | It can be a topic candidate, but it is **not** a citation |
| Topic with 0 valid citations | `brief_available:false`, and the brief returns 409 refuse. It can't be the recs focus. |
| No topic has citations | `recommendations.available:false` (structured refusal, see §4.4). The scan is still 200. |
| Contents fails or is empty for a URL | That citation's quote falls back to its Search snippet. The brief still succeeds. |
| Upstream 401/403 (bad key) | Same 502 generic body. The server logs only `you.com status=<code>`. |
| `/api/gource` before any successful scan | 404 `{"error":"no_scan","message":"Run a scan first."}` |

Upstream response bodies are **never** forwarded to the client.

---

## 4. Contracts

### 4.1 Python module contracts

**`signalfire/you_client.py`** (unchanged from v1)
```python
SEARCH_URL = "https://ydc-index.io/v1/search"      # [VERIFY]
CONTENTS_URL = "https://ydc-index.io/v1/contents"  # [VERIFY]

class EngineUnavailable(Exception): ...   # key missing
class UpstreamError(Exception): ...       # any HTTP/timeout/parse failure; generic message, never key/body

@dataclass(frozen=True)
class Hit:
    url: str; title: str; snippet: str; page_age: str | None; kind: str   # kind: "web" | "news"

def require_key() -> str
def search(query: str, *, count: int = 10, freshness: str = "week") -> list[Hit]
def contents(urls: list[str]) -> dict[str, str]     # keys are ranking.normalize_url(url)
def _call(method: str, url: str, **kw) -> object     # shared try/raise_for_status/json wrapper
```

**`signalfire/ranking.py`**
```python
QUERY_TEMPLATES = ("{niche} trend", "{niche} news", "{niche} meme")   # [DECISION] v2: meme replaces bare niche
MEME_QUERY_INDEX = 2
# HEURISTIC: hand-set weights, not trained or validated
WEIGHTS = {"recency": 0.40, "corroboration": 0.30, "fanout": 0.20, "news": 0.10}
BLOCKLIST = ("nsfw", "porn", "gore", "beheading", "self-harm")
SOCIAL_HOSTS = ("reddit.com", "x.com", "twitter.com", "threads.net", "youtube.com", "tiktok.com")
STOPWORDS = frozenset({"this","that","with","from","your","what","when","over","into","about","says",
                       "they","their","have","will","just","more","than","here","after"})

@dataclass(frozen=True)
class Citation: url: str; title: str; snippet: str

@dataclass
class Candidate:
    key: str; url: str; title: str; snippet: str; page_age: str | None
    is_news: bool; fanout_hits: int; role: str          # role ∈ {"search","news","meme"}
    recency: float = 0.0; corroborators: int = 0; score: float = 0.0
    breakdown: dict = field(default_factory=dict); related_keys: list = field(default_factory=list)

@dataclass
class Topic:
    id: str; title: str; score: float; momentum: str
    breakdown: dict; citations: list; is_news: bool
    role: str = "search"; key: str = ""

class InsufficientSignal(Exception): ...   # .found: int

def queries_for(niche: str) -> list[str]
def normalize_url(url: str) -> str
def host_of(url: str) -> str                         # normalize_url(url).split("/")[0]
def is_social(url: str) -> bool                      # host == h or host.endswith("." + h) for h in SOCIAL_HOSTS
def clean_title(title: str) -> str
def to_citation(hit_or_cand) -> Citation | None
def recency(page_age: str | None, now: datetime) -> float
def title_tokens(title: str, niche: str) -> set[str]
def related(a: set[str], b: set[str]) -> bool        # len(a & b) >= 2
def score(*, recency: float, corroborators: int, fanout_hits: int, queries: int, is_news: bool) -> tuple[float, dict]
def _merge(hits_per_query: list[list[Hit]]) -> list[Candidate]
def eligible(niche: str, hits_per_query: list[list[Hit]], now: datetime) -> list[Candidate]  # merged, unblocked, scored, sorted
def pick(niche: str, cands: list[Candidate]) -> list[Topic]                                 # exactly 3 or raises
def rank(niche: str, hits_per_query, now) -> list[Topic]                                    # = pick(niche, eligible(...))
```

**`signalfire/recs.py`**
```python
NOTE = "Recommendations are rules over cited sources. Not a prediction."
REFUSE_MESSAGE = "No topic has usable sources. SignalFire won't recommend without citations."
TIMING = {  # keyed by recency bucket; [DECISION] window numbers are rule labels, not stats
    1.0: ("next 24h",  "Post within 24 hours. The lead source is under a day old."),
    0.6: ("next 48h",  "Post within 48 hours. The lead source is 1 to 3 days old."),
    0.3: ("this week", "No rush. The lead source is 3 to 7 days old, so bring a new angle."),
    0.0: ("unknown",   "Timing unclear. The lead source has no reliable date; treat it as evergreen."),
}
def format_for(topic: Topic) -> str
def hooks_for(topic: Topic) -> list[dict]        # requires ≥1 citation
def timing_for(topic: Topic) -> dict
def build(topics: list[Topic]) -> dict
```

**`signalfire/gource.py`**
```python
ROLES = ("search", "news", "meme", "rec")
REC_FIELDS = ("focus", "hook", "timing", "content")
WINDOW = timedelta(days=7)
LINE_RE = re.compile(r"^\d{10}\|(search|news|meme|rec)\|[AMD]\|(/[a-z0-9-]+)+$")
def safe_slug(text: str) -> str
def platform(cand: Candidate) -> str             # "social" | "news" | "web"
def build_log(niche: str, cands: list[Candidate], topics: list[Topic], recs: dict, now: datetime) -> list[str]
```

**`signalfire/brief.py`**
```python
DISCLAIMER = "SignalFire surfaces public signals. It does not guarantee virality. Trends are volatile; use your own judgment."
def refusal(topic: Topic) -> dict
def build_brief(topic: Topic, page_text: dict[str, str]) -> dict     # requires len(topic.citations) >= 1
```

### 4.2 Ranking: HEURISTIC [CONTRACT]

> **This score is a hand-set HEURISTIC.** The team picked the weights for the demo. They are not trained, calibrated, or validated, and we make no Precision@K or accuracy claim. The score is relative within one scan and is not a prediction. The API returns `"score_kind": "HEURISTIC"` and `ranking.kind: "HEURISTIC"`, and the UI prints "HEURISTIC" next to every score. `research.md`'s 0.4/0.2/0.2/0.1/0.1 formula needs YouTube/TikTok view velocity, which we don't have. It's spec only.

Rules (each one pinned by a test):

1. **Queries:** `queries_for(n) == [f"{n} trend", f"{n} news", f"{n} meme"]`. Q = `len(hits_per_query)` (3 in the app, even if some slots are `[]` after a failure).
2. **Normalize URL:** Lowercase the scheme and host, strip a leading `www.`, drop the query and fragment, and strip a trailing `/`. Return `host + path` with no scheme. This is `Candidate.key`.
3. **Merge (`_merge`):** Iterate queries in order. Within each query, take web hits in list order, then news hits in list order. Group by `key`.
   - Keep the first-seen `url` and `title`, plus the first non-empty `snippet`.
   - `page_age` is the most recent parseable value across occurrences.
   - `is_news` is true if any occurrence is `news`.
   - `fanout_hits` is the number of **distinct query indices** the key appeared in.
   - **Role [DECISION]:** `"meme"` if the key appeared in query index `MEME_QUERY_INDEX`; else `"news"` if `is_news`; else `"search"`. (Meme beats news beats search.)
   - Output keeps first-seen order.
4. **Blocklist:** `eligible()` drops any candidate whose lowercased cleaned title contains a `BLOCKLIST` entry. Blocked candidates never reach topics, recs, or the Gource log.
5. **Clean title:** Split on the first separator found, in this order: `" | "`, `" - "`, `" — "`. If there is more than one segment and the last has ≤ 4 words, drop the last. Strip, then cut to 90 chars. `Candidate.title` stores the cleaned title.
6. **Tokens:** `re.findall(r"[a-z0-9']+", title.lower())`. Keep tokens with len ≥ 4 that are not in `STOPWORDS` and not tokens of the niche.
7. **Related:** Two candidates are related if they share ≥ 2 tokens. **[VERIFY: tune only the `2` after one live scan, and record it here]**
8. **Corroborators:** For each candidate, count the other eligible candidates related to it. Store their keys in `related_keys`.
9. **Recency** (hours = now − page_age): < 24 (including future) → `1.0`; < 72 → `0.6`; ≤ 168 → `0.3`; older, `None`, or unparseable → `0.0`.
10. **Score:**
    ```
    corroboration = min(corroborators, 3) / 3
    fanout        = (fanout_hits - 1) / (Q - 1)   if Q > 1 else 0.0
    news          = 1.0 if is_news else 0.0
    raw   = 0.40*recency + 0.30*corroboration + 0.20*fanout + 0.10*news
    score = round(raw * 10, 1)                       # 0.0–10.0
    momentum = "high" if score >= 7 else "medium" if score >= 4 else "low"
    breakdown = {"recency":…, "corroboration":…, "fanout":…, "news":…}   # each 0–1, rounded to 2 dp
    ```
11. **Sort and pick (`pick`):** `eligible()` returns candidates sorted by `(-score, key)`. `pick` takes candidates greedily, skipping any related to one already picked. If fewer than 3 are picked, it fills from the skipped ones in sort order. If there are fewer than 3 eligible candidates in total, it raises `InsufficientSignal(found=n)`. Output is in pick order, which is score-descending.
12. **Citations per topic:** The candidate itself (if `to_citation` is valid), then its related candidates with valid citations, sorted by recency descending, then key. Dedupe by key and cap at 3.
13. **Valid citation:** `url` starts with `http://` or `https://`, and `title` and `snippet` are both non-empty after strip.
14. **Topic id:** `"t_" + sha1(f"{niche}|{key}".encode()).hexdigest()[:10]`. `Topic.key = cand.key`, `Topic.role = cand.role`.

### 4.3 Recommendations [CONTRACT]

Recommendations are rules over `topics` (the 3 picked). They never call the engine and never add a URL that isn't in a topic's citations.

- **Focus topic:** The first topic in pick order with `len(citations) >= 1`. If there is none, return the refusal shape below.
- **`format_for(topic)`**, first match wins (shared with the brief, DRY):
  1. Any citation `is_social(url)` → `"Reaction / stitch: respond to the community take and show the source on screen."`
  2. `topic.is_news` → `"Short explainer: what changed this week, with the headline on screen as B-roll."`
  3. `topic.role == "meme"` → `"Meme remix: use the format, add your niche's punchline, credit the source on screen."`
  4. Otherwise → `"Three-point carousel or short: what it is, who it affects, what to do."`
- **`hooks_for(topic)`:** Let `c1 = topic.citations[0]`. The result is a list of dicts `{"text": str, "topic_id": topic.id, "source_url": c1.url}`, in this order:
  1. `f"Everyone's about to ask about {title}. Here's what the sources actually say."`
  2. `f"\"{c1.snippet}\" Here's what that means for you."`. **Only** if `len(c1.snippet) <= 140` and `"viral" not in c1.snippet.lower()`; otherwise omit it.
  3. `f"I read the sources on {title} so you don't have to."`
- **`timing_for(topic)`:** `r = topic.breakdown.get("recency", 0.0)`, then look up `TIMING[r]` (falling back to `0.0` for an unknown value). Returns `{"window": w, "label": l, "topic_id": topic.id, "basis": "HEURISTIC recency bucket of the lead source"}`.
- **`build(topics)`** when a focus exists:
  ```json
  {"available": true,
   "focus": {"topic_id": "…", "title": "…", "score": 7.0, "momentum": "high",
             "reason": "Highest HEURISTIC score among topics with sources (2 cited)."},
   "hooks": [{"text": "…", "topic_id": "…", "source_url": "https://…"}],
   "timing": {"window": "next 24h", "label": "…", "topic_id": "…", "basis": "HEURISTIC recency bucket of the lead source"},
   "content_format": {"format": "…", "topic_id": "…"},
   "note": "Recommendations are rules over cited sources. Not a prediction."}
  ```
- **Refusal shape** (no topic has citations):
  `{"available": false, "message": REFUSE_MESSAGE, "focus": null, "hooks": [], "timing": null, "content_format": null, "note": NOTE}`
- **Invariants:** Every `source_url` is a citation URL of the focus topic. Digits in hook texts appear in the topic title or `c1.snippet`. The word "viral" appears nowhere in `build()` output.

### 4.4 Gource-format trend log [CONTRACT]

This is Gource's **custom log** format, `unix|user|type|path`. "user" is the signal **role**. It is a synthetic replay of *this scan's* You.com results. It is **not** git history and **not** a live hunt. The UI plays it as an SVG tree; nothing runs the Gource OpenGL binary.

- **Line:** `f"{unix}|{role}|{type}|{path}"`. `unix` is an int of seconds (10 digits). `role` ∈ `ROLES`. `type` ∈ `A` (added), `M` (modified = ranked into the top 3), `D` (deleted = dropped). Every line must match `LINE_RE`.
- **Path:** `"/" + safe_slug(niche) + "/" + platform + "/" + safe_slug(title)`. For recs it's `"/" + safe_slug(niche) + "/rec/" + field`. It always starts with `/`, and each segment matches `[a-z0-9-]+`.
- **`platform(cand)`:** `"social"` if `is_social(cand.url)`; else `"news"` if `cand.is_news`; else `"web"`. The host itself **never** appears.
- **`safe_slug(text)` [DECISION]:**
  1. Split on whitespace. Drop any token where `re.search(r"://|@|\w\.\w|:\d", tok)` matches or `tok.lower().startswith("www")`. This removes URLs, emails, IPs, ports, and bare domains, including dotted tokens like `2.0`, which is an accepted loss.
  2. Join with a space, lowercase, and apply `re.sub(r"[^a-z0-9]+", "-", s).strip("-")`.
  3. If `len > 40`: `s = s[:41].rsplit("-", 1)[0][:40].strip("-")` (cuts at a word boundary).
  4. If the result is empty, use `"untitled"`.
- **Events:**
  - **A**: One per eligible candidate, with `role = cand.role` and the candidate path. `ts = clamp(parse(page_age), now − WINDOW, now)`, or `now` if `page_age` is `None` or unparseable.
  - **M**: One per picked topic, in pick order. `ts = now`, `role = topic.role`, and the path is that topic's candidate path (looked up by `topic.key`).
  - **D**: One per eligible candidate that is neither picked nor cited by any picked topic, in candidate order. `ts = now`, `role = cand.role`.
  - **rec A**: Only if `recs["available"]`. One per field in `REC_FIELDS`, with `ts = now`, `role = "rec"`, and path `/<niche>/rec/<field>`.
- **Sort [CONTRACT]:** Sort by `(ts, phase, index)` where phase is A=0, M=1, D=2, rec=3, and index is the order within the phase. The output is ascending by `unix`.
- Duplicate paths are allowed (Gource treats them as the same file).

### 4.5 Brief [CONTRACT]

- **Refuse when `len(topic.citations) == 0`**, before any engine call. Refusing needs no key.
- `confidence` depends on citation count only: 1 → `"low"`, 2 → `"medium"`, 3 → `"medium-high"`. **Never `"high"`**, because nothing is calibrated. `confidence_basis = f"{n} source"` + (`"s"` if n ≠ 1).
- `format_idea = recs.format_for(topic)`.
- `hook = recs.hooks_for(topic)[0]["text"]`.
- `timing_tip = recs.timing_for(topic)["label"]`.
- `content_tip = f"Put source [1] on screen ({host_of(c1.url)}) while you read its quote, then add your own angle."`
- `why_now`: one `{"quote": str, "cite": i}` per citation, with `i` 1-based. The quote is the first sentence of `page_text[normalize_url(c.url)]` (split on `(?<=[.!?])\s+`) that is 40–240 chars, contains a token from `title_tokens(topic.title, "")`, and does not contain "viral". If there is none, use `c.snippet` verbatim.
- `disclaimer` is the exact `DISCLAIMER`.
- **No invented content:** Every URL in the brief is in `topic.citations`. Digits in `format_idea + hook` appear in `topic.title`. "viral" does not appear in `format_idea`, `hook`, `timing_tip`, or `content_tip`.

### 4.6 HTTP API [CONTRACT]

Routes: `GET /`, `POST /api/scan`, `POST /api/brief`, `POST /api/gource`. No others, no CORS, bound to `127.0.0.1` only.

**`POST /api/scan`** body `{"niche": str}`. The model is `niche: str | None = None`, so a missing field returns **400**, not FastAPI's 422.

Check order: niche (400) → key (503) → upstream (502) → signal (422) → 200.

| Case | Status | Body |
|---|---|---|
| `niche` missing, empty, or whitespace | 400 | `{"error":"invalid_niche","message":"Enter a niche, e.g. personal finance for Gen Z."}` |
| `len(niche.strip()) > 80` | 400 | `{"error":"invalid_niche","message":"Keep the niche under 80 characters."}` |
| no key | 503 | `{"error":"engine_unavailable","message":"Live search is not configured."}` |
| all 3 queries failed | 502 | `{"error":"upstream_error","message":"Live search failed. Try again."}` |
| < 3 eligible | 422 | `{"error":"insufficient_signal","message":"Not enough fresh sources for this niche. Try a broader niche.","found":N}` |
| OK | 200 | below |

```json
{
  "niche": "personal finance for Gen Z",
  "generated_at": "2026-11-22T12:00:00Z",
  "engine": {"provider": "you.com", "endpoints": ["search"],
             "queries": ["personal finance for Gen Z trend", "personal finance for Gen Z news", "personal finance for Gen Z meme"]},
  "ranking": {"kind": "HEURISTIC", "weights": {"recency": 0.4, "corroboration": 0.3, "fanout": 0.2, "news": 0.1},
              "note": "Hand-set heuristic. Not a prediction."},
  "topics": [
    {"id": "t_ab12cd34ef", "title": "…", "score": 7.0, "score_kind": "HEURISTIC", "momentum": "high", "role": "news",
     "breakdown": {"recency": 1.0, "corroboration": 0.33, "fanout": 0.5, "news": 1.0},
     "citations": [{"url": "https://…", "title": "…", "snippet": "…"}],
     "brief_available": true}
  ],
  "recommendations": { "available": true, "focus": {…}, "hooks": […], "timing": {…}, "content_format": {…}, "note": "…" },
  "gource_log": ["1795348800|news|A|/personal-finance-for-gen-z/news/…", "…"]
}
```
`topics` has exactly 3 entries, sorted by score descending. `generated_at` is `clock()` as ISO `Z`. On 200 only, the app stores `app.state.topics[id] = Topic` for each topic (which accumulate across scans) and **overwrites** `app.state.last_gource = {"niche", "generated_at", "lines"}`. Failed scans (400/502/503/422) leave both untouched.

**`POST /api/brief`** body `{"topic_id": str}`. Order: 404 → 409 → 503 → 200. A Contents failure is not an error.

| Case | Status | Body |
|---|---|---|
| unknown or missing id | 404 | `{"error":"unknown_topic","message":"Run a scan first."}` |
| 0 citations | **409** | `{"error":"insufficient_citations","refused":true,"topic_id":"…","title":"…","message":"No usable sources for this topic. SignalFire won't write a brief without citations.","citations":[]}` |
| no key (cites exist) | 503 | engine_unavailable body |
| OK | 200 | below |

```json
{"topic_id": "…", "title": "…", "refused": false,
 "format_idea": "…", "hook": "…", "timing_tip": "…", "content_tip": "…",
 "why_now": [{"quote": "…", "cite": 1}],
 "confidence": "medium", "confidence_basis": "2 sources",
 "citations": [{"url": "…", "title": "…", "snippet": "…"}],
 "engine": {"provider": "you.com", "endpoints": ["contents"]},
 "disclaimer": "SignalFire surfaces public signals. It does not guarantee virality. Trends are volatile; use your own judgment."}
```

**`POST /api/gource`** takes no body (anything sent is ignored).

| Case | Status | Body |
|---|---|---|
| no successful scan yet in this process | 404 | `{"error":"no_scan","message":"Run a scan first."}` |
| OK | 200 | `{"niche":"…","generated_at":"…","format":"gource-custom-log","roles":["search","news","meme","rec"],"lines":[…]}` (same `lines` as the last 200 scan's `gource_log`) |

**`GET /`** returns `FileResponse(signalfire/static/index.html)`.

---

## 5. Bite-sized TDD tasks (tests first, pytest)

**Order is priority.** P0 is T0–T25. P1 is T26–T27 plus the UI download button. Every task runs the same loop: write the test → run it → **see it fail for the stated reason** → minimal impl → green → run the full suite.

### Fixture spec

All fixtures use RFC-reserved `example.*` domains.

**`tests/fixtures/search_ok.json`**: `{"results": {"web": [W1..W6], "news": [N1, N2]}}` (same as v1)

| id | kind | url | title | snippet (`snippets:[…]`) | page_age |
|---|---|---|---|---|---|
| W1 | web | `https://www.example-news.com/money/loud-budgeting-gen-z?utm_source=x` | `Loud Budgeting Is Taking Over Gen Z Money Talk \| Example News` | "Creators are posting the purchases they refuse to make." | `2026-11-21T18:00:00Z` |
| W2 | web | `https://blog.example.org/loud-budgeting-explained` | `Loud Budgeting, Explained: Why Gen Z Says It Out Loud` | "The trend reframes saying no as a flex." | `2026-11-20T12:00:00Z` |
| W3 | web | `https://www.example.com/cash-stuffing-comeback` | `The Cash Stuffing Comeback Nobody Expected` | "Envelope budgeting videos are back in feeds." | `2026-11-19T20:00:00Z` |
| W4 | web | `https://forum.example.net/t/cash-stuffing-envelopes` | `Cash stuffing envelopes thread` | **`snippets: []`, `description: ""`** | *(absent)* |
| W5 | web | `https://www.example.com/roth-ira-teens` | `Roth IRA for Teens: New Contribution Rules` | "What changed for teen contribution limits." | `2026-11-17T12:00:00Z` |
| W6 | web | `https://www.example.com/nsfw-finance` | `NSFW finance memes` | "…" | `2026-11-22T10:00:00Z` |
| N1 | news | `https://news.example.com/2026/11/22/bnpl-holiday` | `Buy Now, Pay Later Use Spikes Ahead of Holidays` | "Shoppers lean on installment plans this season." | `2026-11-22T09:00:00Z` |
| N2 | news | `https://example-news.com/money/loud-budgeting-gen-z` | `Loud budgeting goes mainstream` | "Newsrooms pick up the loud budgeting story." | `2026-11-21T20:00:00Z` |

**`tests/fixtures/search_meme.json`**: `{"results": {"web": [M1], "news": []}}`

| id | kind | url | title | snippet | page_age |
|---|---|---|---|---|---|
| M1 | web | `https://memes.example.com/budget-cat` | `The Budget Cat Meme Is Everywhere` | "Creators pair the cat with their spending confessions." | `2026-11-22T06:00:00Z` |

**`tests/fixtures/contents_ok.json`**: a list with one object:
`{"url":"https://www.example-news.com/money/loud-budgeting-gen-z?utm_source=x","title":"…","markdown":"Short. Loud budgeting means saying no to spending, out loud, on camera. More text follows here."}`

**Golden A: unit, single query list** (`rank(niche, [fixture_hits()], NOW)`, Q = 1, fanout = 0). Same as v1:

| cand | recency | corroborators | news | role | score |
|---|---|---|---|---|---|
| W1+N2 merged (page_age 11-21T20:00) | 1.0 | 1 (W2) | yes | news | **6.0** |
| N1 | 1.0 | 0 | yes | news | **5.0** |
| W2 | 0.6 | 1 | no | search | 3.4, skipped (related to W1; it sorts before W3 by key) |
| W3 | 0.6 | 1 (W4) | no | search | **3.4** |
| W5 | 0.3 | 0 | no | search | 1.2 |
| W4 | 0.0 | 1 | no | search | 1.0 |
| W6 | blocked | | | | |

Topics: `Loud Budgeting Is Taking Over Gen Z Money Talk` (2 cites: W1, W2), `Buy Now, Pay Later Use Spikes Ahead of Holidays` (1: N1), `The Cash Stuffing Comeback Nobody Expected` (1: W3; W4 has no snippet).

**Golden B: app, 3 queries** (`trend`/`news` → `search_ok.json`, `meme` → `search_meme.json`, Q = 3). W-items and N-items appear in 2 queries (fanout 0.5, +1.0 score). M1 appears in 1 query.

| cand | recency | corr | fanout | news | role | score |
|---|---|---|---|---|---|---|
| W1+N2 | 1.0 | 1 | 0.5 | yes | news | **7.0** (high) |
| N1 | 1.0 | 0 | 0.5 | yes | news | **6.0** (medium) |
| W2 | 0.6 | 1 | 0.5 | no | search | 4.4, skipped (related to W1) |
| W3 | 0.6 | 1 | 0.5 | no | search | **4.4** (medium) |
| M1 | 1.0 | 0 | 0.0 | no | meme | 4.0 (not picked → D in log) |
| W5 | 0.3 | 0 | 0.5 | no | search | 2.2 |
| W4 | 0.0 | 1 | 0.5 | no | search | 2.0 |

If your arithmetic disagrees, **re-derive by hand from §4.2. Do not tweak weights, thresholds, or fixtures to make a test pass.**

### T0: `tests/conftest.py` (write first)
```python
import json, pathlib, pytest, httpx, respx
from datetime import datetime, timezone
from fastapi.testclient import TestClient

FIX = pathlib.Path(__file__).parent / "fixtures"
SENTINEL = "SENTINEL-KEY-123"          # obviously fake; never a real key
NOW = datetime(2026, 11, 22, 12, tzinfo=timezone.utc)
NICHE = "personal finance for Gen Z"

def load(name): return json.loads((FIX / name).read_text())

def u(iso): return int(datetime.fromisoformat(iso.replace("Z", "+00:00")).timestamp())

def pytest_configure(config):
    config.addinivalue_line("markers", "live: hits real You.com; needs RUN_LIVE=1 and YOU_API_KEY")

@pytest.fixture
def key(monkeypatch):
    monkeypatch.setenv("YOU_API_KEY", SENTINEL); return SENTINEL

@pytest.fixture
def nokey(monkeypatch):
    monkeypatch.delenv("YOU_API_KEY", raising=False)

def _search(request):
    q = request.url.params.get("query", "")
    return httpx.Response(200, json=load("search_meme.json" if q.endswith(" meme") else "search_ok.json"))

@pytest.fixture
def you(key):
    from signalfire import you_client
    with respx.mock(assert_all_called=False) as m:
        # [VERIFY] TestClient uses its own transport and should not be intercepted.
        # If it is, add: m.route(host="testserver").pass_through()
        m.get(you_client.SEARCH_URL).mock(side_effect=_search)
        m.post(you_client.CONTENTS_URL).mock(return_value=httpx.Response(200, json=load("contents_ok.json")))
        yield m

@pytest.fixture
def client():
    from signalfire.app import create_app
    return TestClient(create_app(clock=lambda: NOW))

def fixture_hits(name="search_ok.json"):
    from signalfire.you_client import Hit
    res, out = load(name)["results"], []
    for kind in ("web", "news"):
        for it in res.get(kind, []):
            sn = (it.get("snippets") or [""])[0] or it.get("description", "")
            out.append(Hit(it["url"], it["title"], sn, it.get("page_age"), kind))
    return out
```
Command: `python -m pytest -q` → expected: `ModuleNotFoundError: No module named 'signalfire'`. Impl: add `signalfire/__init__.py`, `tests/__init__.py`, `requirements.txt`, then run `pip install -r requirements.txt`.

---

### T1: missing key fails closed, no HTTP [you_client]
```python
# tests/test_you_client.py
import pytest, httpx, respx
from signalfire import you_client as yc
from tests.conftest import load, SENTINEL

def test_missing_key_raises_before_http(nokey):
    with respx.mock(assert_all_mocked=True) as m:
        with pytest.raises(yc.EngineUnavailable):
            yc.search("anything")
        assert m.calls.call_count == 0
```
Run: `python -m pytest -q tests/test_you_client.py::test_missing_key_raises_before_http` → fails with `ImportError: cannot import name 'you_client'`.
Impl: `require_key()` reads `os.environ.get("YOU_API_KEY", "").strip()` and raises `EngineUnavailable("engine_unavailable")` when it's empty. `search()` and `contents()` call it first.

### T2: search authenticates, sends params, parses web + news
```python
def test_search_parses_and_authenticates(key):
    with respx.mock as m:
        r = m.get(yc.SEARCH_URL).mock(return_value=httpx.Response(200, json=load("search_ok.json")))
        hits = yc.search("personal finance for Gen Z trend")
    req = r.calls[0].request
    assert req.headers["X-API-Key"] == SENTINEL
    assert req.url.params["query"] == "personal finance for Gen Z trend"
    assert req.url.params["freshness"] == "week" and req.url.params["count"] == "10"
    assert len(hits) == 8 and [h.kind for h in hits] == ["web"] * 6 + ["news"] * 2
    w4 = next(h for h in hits if "forum.example.net" in h.url)
    assert w4.snippet == "" and w4.page_age is None

def test_search_tolerates_missing_sections(key):
    with respx.mock as m:
        m.get(yc.SEARCH_URL).mock(return_value=httpx.Response(200, json={"results": {"web": []}}))
        assert yc.search("x") == []
```
Expected fail: `AttributeError: … has no attribute 'SEARCH_URL'`.
Impl: Use `httpx.Client(timeout=8.0)` and `GET` with params `query, count, freshness, country=US`. Parse with `.get()` defaults. Snippet rule as in §3.

### T3: upstream failures are generic, never leak the key or body
```python
@pytest.mark.parametrize("resp", [
    httpx.Response(401, text="bad key SENTINEL-KEY-123"),
    httpx.Response(500, text="boom"),
    httpx.Response(200, text="not json")])
def test_upstream_errors_are_generic(key, resp):
    with respx.mock as m:
        m.get(yc.SEARCH_URL).mock(return_value=resp)
        with pytest.raises(yc.UpstreamError) as e:
            yc.search("x")
    assert SENTINEL not in str(e.value) and "boom" not in str(e.value)

def test_timeout_is_upstream_error(key):
    with respx.mock as m:
        m.get(yc.SEARCH_URL).mock(side_effect=httpx.ReadTimeout("t"))
        with pytest.raises(yc.UpstreamError):
            yc.search("x")
```
Expected fail: `httpx.HTTPStatusError` leaks out, or nothing is raised.
Impl: `_call()` wraps the request, `raise_for_status()`, and `.json()`. On `httpx.HTTPError` or `ValueError` it does `raise UpstreamError("upstream_error") from None` and logs only the status code.

### T4: contents maps normalized URL to text
```python
def test_contents_maps_normalized_urls(key):
    with respx.mock as m:
        r = m.post(yc.CONTENTS_URL).mock(return_value=httpx.Response(200, json=load("contents_ok.json")))
        out = yc.contents(["https://www.example-news.com/money/loud-budgeting-gen-z?utm_source=x"])
    assert r.calls[0].request.headers["X-API-Key"] == SENTINEL
    assert out["example-news.com/money/loud-budgeting-gen-z"].startswith("Short. Loud budgeting")

def test_contents_failure_is_upstream_error(key):
    with respx.mock as m:
        m.post(yc.CONTENTS_URL).mock(return_value=httpx.Response(503))
        with pytest.raises(yc.UpstreamError):
            yc.contents(["https://example.com/a"])
```
Expected fail: `AttributeError: … 'contents'`. Impl: Send the POST body from §3 through `_call`. Text is the first non-empty of `markdown/text/html/content`.

### T5: citation validity, URL normalization, title cleaning [ranking]
```python
# tests/test_ranking.py
import pytest
from signalfire import ranking as rk
from signalfire.you_client import Hit
from tests.conftest import NOW, NICHE, fixture_hits

def test_to_citation_requires_url_title_snippet():
    assert rk.to_citation(Hit("https://a.example/x", "T", "s", None, "web")) == rk.Citation("https://a.example/x", "T", "s")
    assert rk.to_citation(Hit("https://a.example/x", "T", "  ", None, "web")) is None
    assert rk.to_citation(Hit("ftp://a.example/x", "T", "s", None, "web")) is None

def test_normalize_url_and_host():
    assert rk.normalize_url("HTTPS://WWW.Example.com/a/b/?q=1#f") == "example.com/a/b"
    assert rk.host_of("https://www.reddit.com/r/x") == "reddit.com"
    assert rk.is_social("https://old.reddit.com/r/x") and not rk.is_social("https://notreddit.com/x")

def test_clean_title_drops_site_suffix():
    assert rk.clean_title("Loud Budgeting Is Taking Over Gen Z Money Talk | Example News") == "Loud Budgeting Is Taking Over Gen Z Money Talk"
    assert rk.clean_title("Roth IRA for Teens: New Contribution Rules") == "Roth IRA for Teens: New Contribution Rules"
```
Expected fail: `ImportError: cannot import name 'ranking'`. Impl: §4.2 items 2, 5, and 13, using `urllib.parse`.

### T6: recency buckets
```python
@pytest.mark.parametrize("age,exp", [
    ("2026-11-22T09:00:00Z", 1.0), ("2026-11-23T00:00:00Z", 1.0),
    ("2026-11-20T12:00:00Z", 0.6), ("2026-11-19T12:00:00Z", 0.3),
    ("2026-11-10T00:00:00Z", 0.0), (None, 0.0), ("garbage", 0.0)])
def test_recency(age, exp):
    assert rk.recency(age, NOW) == exp
```
Impl: `datetime.fromisoformat(age.replace("Z", "+00:00"))`. `ValueError`, `TypeError`, or `None` → 0.0.

### T7: HEURISTIC score formula, golden values
```python
def test_score_golden():
    s, b = rk.score(recency=1.0, corroborators=3, fanout_hits=3, queries=3, is_news=True)
    assert s == 10.0 and b == {"recency": 1.0, "corroboration": 1.0, "fanout": 1.0, "news": 1.0}
    assert rk.score(recency=0.6, corroborators=1, fanout_hits=1, queries=3, is_news=False)[0] == 3.4
    assert rk.score(recency=1.0, corroborators=0, fanout_hits=1, queries=1, is_news=False)[0] == 4.0
    assert rk.score(recency=1.0, corroborators=1, fanout_hits=2, queries=3, is_news=True)[0] == 7.0

def test_weights_are_documented_heuristic():
    assert rk.WEIGHTS == {"recency": 0.40, "corroboration": 0.30, "fanout": 0.20, "news": 0.10}
    assert abs(sum(rk.WEIGHTS.values()) - 1.0) < 1e-9
```
Impl: §4.2 item 10 exactly, with the `# HEURISTIC` comment above `WEIGHTS`.

### T8: queries, merge, fan-out, roles
```python
def test_queries_for():
    assert rk.queries_for("gen z money") == ["gen z money trend", "gen z money news", "gen z money meme"]

def test_merge_fanout_order_and_roles():
    a = Hit("https://www.example.com/x?u=1", "Alpha Bravo Charlie", "s", None, "web")
    b = Hit("https://example.com/x", "Alpha Bravo Charlie", "s", None, "news")
    c = Hit("https://example.com/y", "Delta Echo Foxtrot", "s", None, "web")
    n = Hit("https://example.com/n", "Golf Hotel India", "s", None, "news")
    merged = rk._merge([[a, n, c], [b], [a]])          # index 2 is the meme query
    assert [x.key for x in merged] == ["example.com/x", "example.com/y", "example.com/n"]  # web before news
    m = {x.key: x for x in merged}
    assert m["example.com/x"].fanout_hits == 3 and m["example.com/x"].is_news and m["example.com/x"].role == "meme"
    assert m["example.com/y"].role == "search" and m["example.com/y"].fanout_hits == 1
    assert m["example.com/n"].role == "news"
```
Expected fail: `AttributeError: … '_merge'`. Impl: §4.2 items 1 and 3.

### T9: rank picks exactly 3, distinct, blocklist applied (Golden A)
```python
def test_rank_golden():
    topics = rk.rank(NICHE, [fixture_hits()], NOW)
    assert [t.title for t in topics] == [
        "Loud Budgeting Is Taking Over Gen Z Money Talk",
        "Buy Now, Pay Later Use Spikes Ahead of Holidays",
        "The Cash Stuffing Comeback Nobody Expected"]
    assert [t.score for t in topics] == [6.0, 5.0, 3.4]
    assert [t.role for t in topics] == ["news", "news", "search"]
    assert [len(t.citations) for t in topics] == [2, 1, 1]
    assert all(t.id.startswith("t_") and len(t.id) == 12 for t in topics)
    urls = {c.url for t in topics for c in t.citations}
    assert "https://forum.example.net/t/cash-stuffing-envelopes" not in urls

def test_eligible_drops_blocklist_and_sorts():
    cands = rk.eligible(NICHE, [fixture_hits()], NOW)
    assert not any("nsfw" in c.title.lower() for c in cands) and len(cands) == 6
    assert [c.score for c in cands] == sorted((c.score for c in cands), reverse=True)

def test_rank_insufficient_signal():
    with pytest.raises(rk.InsufficientSignal) as e:
        rk.rank("x", [fixture_hits()[:1]], NOW)
    assert e.value.found == 1

def test_rank_fills_when_all_related():
    hs = [Hit(f"https://e.example/{i}", f"Loud Budgeting Story {i}", "s", "2026-11-22T10:00:00Z", "web") for i in range(3)]
    assert len(rk.rank("x", [hs], NOW)) == 3
```
Impl: §4.2 items 4–14. `related()` is shared between corroboration and distinctness (DRY).

### T10: recommendations, golden (focus, hooks, timing, content) [recs]
```python
# tests/test_recs.py
import re, json
from signalfire import ranking as rk, recs
from tests.conftest import NOW, NICHE, fixture_hits

TITLE = "Loud Budgeting Is Taking Over Gen Z Money Talk"

def test_recs_golden():
    topics = rk.rank(NICHE, [fixture_hits()], NOW)
    r = recs.build(topics)
    assert r["available"] is True and r["note"] == recs.NOTE
    assert r["focus"]["topic_id"] == topics[0].id and r["focus"]["title"] == TITLE
    assert r["focus"]["reason"] == "Highest HEURISTIC score among topics with sources (2 cited)."
    assert [h["text"] for h in r["hooks"]] == [
        f"Everyone's about to ask about {TITLE}. Here's what the sources actually say.",
        '"Creators are posting the purchases they refuse to make." Here\'s what that means for you.',
        f"I read the sources on {TITLE} so you don't have to."]
    assert {h["source_url"] for h in r["hooks"]} == {topics[0].citations[0].url}
    assert r["timing"]["window"] == "next 24h"
    assert r["timing"]["label"] == "Post within 24 hours. The lead source is under a day old."
    assert r["content_format"]["format"].startswith("Short explainer")

def test_format_rules_order():
    t = lambda cites, news=False, role="search": rk.Topic("t_x", "T", 1.0, "low", {}, cites, news, role)
    social = rk.Citation("https://www.reddit.com/r/x/1", "t", "s")
    web = rk.Citation("https://e.example/a", "t", "s")
    assert recs.format_for(t([social], news=True)).startswith("Reaction / stitch")
    assert recs.format_for(t([web], news=True)).startswith("Short explainer")
    assert recs.format_for(t([web], role="meme")).startswith("Meme remix")
    assert recs.format_for(t([web])).startswith("Three-point")

def test_timing_buckets():
    for r, w in [(1.0, "next 24h"), (0.6, "next 48h"), (0.3, "this week"), (0.0, "unknown")]:
        assert recs.timing_for(rk.Topic("t_x", "T", 1.0, "low", {"recency": r}, [], False))["window"] == w
    assert recs.timing_for(rk.Topic("t_x", "T", 1.0, "low", {}, [], False))["window"] == "unknown"
```
Expected fail: `ImportError: cannot import name 'recs'`. Impl: §4.3.

### T11: recs refuse without cites, skip uncited focus, no invented URLs/digits/"viral"
```python
def _t(id_, cites, score):
    return rk.Topic(id_, f"Topic {id_}", score, "low", {"recency": 1.0}, cites, False)

C = rk.Citation("https://e.example/a", "A", "Short snippet.")

def test_recs_refuse_when_no_citations():
    r = recs.build([_t("t_a", [], 9.0), _t("t_b", [], 5.0), _t("t_c", [], 1.0)])
    assert r == {"available": False, "message": recs.REFUSE_MESSAGE, "focus": None, "hooks": [],
                 "timing": None, "content_format": None, "note": recs.NOTE}
    assert "http" not in json.dumps(r)

def test_focus_skips_uncited_topic():
    r = recs.build([_t("t_a", [], 9.0), _t("t_b", [C], 5.0), _t("t_c", [], 1.0)])
    assert r["focus"]["topic_id"] == "t_b"

def test_snippet_hook_omitted_when_long_or_viral():
    long_c = rk.Citation("https://e.example/a", "A", "x" * 141)
    viral_c = rk.Citation("https://e.example/a", "A", "This went viral overnight.")
    assert len(recs.hooks_for(_t("t_a", [long_c], 1.0))) == 2
    assert len(recs.hooks_for(_t("t_a", [viral_c], 1.0))) == 2

def test_recs_invariants_on_golden():
    topics = rk.rank(NICHE, [fixture_hits()], NOW)
    r = recs.build(topics); s = json.dumps(r)
    allowed = {c.url for t in topics for c in t.citations}
    assert set(re.findall(r"https?://[^\s\"']+", s)) <= allowed
    assert "viral" not in s.lower()
    c1 = topics[0].citations[0]
    for h in r["hooks"]:
        assert set(re.findall(r"\d+", h["text"])) <= set(re.findall(r"\d+", topics[0].title + c1.snippet))
```

### T12: Gource slug safety [gource]
```python
# tests/test_gource.py
import re
from signalfire import gource as gs, ranking as rk, recs
from tests.conftest import NOW, NICHE, fixture_hits, u

IPV4 = re.compile(r"\d{1,3}(\.\d{1,3}){3}")

def test_safe_slug_basic():
    assert gs.safe_slug(NICHE) == "personal-finance-for-gen-z"
    assert gs.safe_slug("Buy Now, Pay Later Use Spikes Ahead of Holidays") == "buy-now-pay-later-use-spikes-ahead-of"
    assert gs.safe_slug("Loud Budgeting Is Taking Over Gen Z Money Talk") == "loud-budgeting-is-taking-over-gen-z"
    assert gs.safe_slug("!!!") == "untitled"

def test_safe_slug_strips_urls_emails_ips_ports():
    s = gs.safe_slug("Watch https://evil.example.com:8080/x now, email me@evil.example or 10.0.0.1:22 www.evil.example")
    assert s == "watch-now-email-or"
    for bad in ("http", "www", "evil", "example", "@", ":", "."):
        assert bad not in s
    assert not IPV4.search(s)
```
Expected fail: `ImportError: cannot import name 'gource'`. Impl: `safe_slug` per §4.4.

### T13: Gource log, golden lines, format, sort, honesty
```python
def _golden_inputs():
    cands = rk.eligible(NICHE, [fixture_hits()], NOW)
    topics = rk.pick(NICHE, cands)
    return cands, topics, recs.build(topics)

def test_build_log_golden():
    cands, topics, r = _golden_inputs()
    P = "/personal-finance-for-gen-z"; now = u("2026-11-22T12:00:00Z")
    assert gs.build_log(NICHE, cands, topics, r, NOW) == [
        f"{u('2026-11-17T12:00:00Z')}|search|A|{P}/web/roth-ira-for-teens-new-contribution",
        f"{u('2026-11-19T20:00:00Z')}|search|A|{P}/web/the-cash-stuffing-comeback-nobody",
        f"{u('2026-11-20T12:00:00Z')}|search|A|{P}/web/loud-budgeting-explained-why-gen-z-says",
        f"{u('2026-11-21T20:00:00Z')}|news|A|{P}/news/loud-budgeting-is-taking-over-gen-z",
        f"{u('2026-11-22T09:00:00Z')}|news|A|{P}/news/buy-now-pay-later-use-spikes-ahead-of",
        f"{now}|search|A|{P}/web/cash-stuffing-envelopes-thread",
        f"{now}|news|M|{P}/news/loud-budgeting-is-taking-over-gen-z",
        f"{now}|news|M|{P}/news/buy-now-pay-later-use-spikes-ahead-of",
        f"{now}|search|M|{P}/web/the-cash-stuffing-comeback-nobody",
        f"{now}|search|D|{P}/web/cash-stuffing-envelopes-thread",
        f"{now}|search|D|{P}/web/roth-ira-for-teens-new-contribution",
        f"{now}|rec|A|{P}/rec/focus",
        f"{now}|rec|A|{P}/rec/hook",
        f"{now}|rec|A|{P}/rec/timing",
        f"{now}|rec|A|{P}/rec/content"]

def test_log_lines_are_safe_and_sorted():
    cands, topics, r = _golden_inputs()
    lines = gs.build_log(NICHE, cands, topics, r, NOW)
    assert all(gs.LINE_RE.match(l) for l in lines)
    ts = [int(l.split("|")[0]) for l in lines]
    assert ts == sorted(ts)
    for l in lines:
        path = l.split("|")[3]
        assert path.startswith("/") and "://" not in path and "@" not in path and not IPV4.search(path)
        assert not re.search(r":\d", path) and "example" not in path
    assert not any("nsfw" in l for l in lines)                 # blocked never logged

def test_no_rec_lines_when_recs_refused():
    cands, topics, _ = _golden_inputs()
    refused = recs.build([rk.Topic("t", "T", 1.0, "low", {}, [], False)] * 3)
    assert not any("|rec|" in l for l in gs.build_log(NICHE, cands, topics, refused, NOW))

def test_old_page_age_clamped_to_window():
    from signalfire.you_client import Hit
    hs = [Hit(f"https://e.example/{i}", t, "s", "2026-01-01T00:00:00Z", "web")
          for i, t in enumerate(["Alpha Bravo", "Charlie Delta", "Echo Foxtrot"])]
    cands = rk.eligible("x", [hs], NOW); topics = rk.pick("x", cands)
    first = int(gs.build_log("x", cands, topics, recs.build(topics), NOW)[0].split("|")[0])
    assert first == u("2026-11-15T12:00:00Z")
```
Expected fail: `AttributeError: … 'build_log'`. Impl: §4.4. Look up a topic's candidate by `topic.key`, and the cited set by `rk.normalize_url(c.url)`. Assert `LINE_RE` on each line before returning (fail loud).

### T14: brief refuses with zero citations [brief]
```python
# tests/test_brief.py
import re
from signalfire import brief as br, ranking as rk

def topic(cites, title="Loud Budgeting Is Taking Over Gen Z Money Talk", is_news=True):
    return rk.Topic(id="t_0000000000", title=title, score=6.0, momentum="medium",
                    breakdown={"recency": 1.0}, citations=cites, is_news=is_news)

def test_refusal_shape_has_no_sources():
    b = br.refusal(topic([]))
    assert b["refused"] is True and b["error"] == "insufficient_citations"
    assert b["citations"] == [] and "http" not in str(b)
```
Expected fail: `ImportError: cannot import name 'brief'`.

### T15: brief with cites: disclaimer, hook, timing, content tip, quotes, only known URLs
```python
C1 = rk.Citation("https://www.example-news.com/money/loud-budgeting-gen-z?utm_source=x", "Loud Budgeting…", "Creators are posting the purchases they refuse to make.")
C2 = rk.Citation("https://blog.example.org/loud-budgeting-explained", "Loud Budgeting, Explained", "The trend reframes saying no as a flex.")
TEXT = {"example-news.com/money/loud-budgeting-gen-z": "Short. Loud budgeting means saying no to spending, out loud, on camera. More text follows here."}
TITLE = "Loud Budgeting Is Taking Over Gen Z Money Talk"

def test_brief_with_cites():
    b = br.build_brief(topic([C1, C2]), TEXT)
    assert b["refused"] is False
    assert b["disclaimer"] == "SignalFire surfaces public signals. It does not guarantee virality. Trends are volatile; use your own judgment."
    assert b["hook"] == f"Everyone's about to ask about {TITLE}. Here's what the sources actually say."
    assert b["timing_tip"] == "Post within 24 hours. The lead source is under a day old."
    assert b["content_tip"] == "Put source [1] on screen (example-news.com) while you read its quote, then add your own angle."
    assert b["format_idea"].startswith("Short explainer")
    assert b["confidence"] == "medium" and b["confidence_basis"] == "2 sources"
    assert b["why_now"] == [
        {"quote": "Loud budgeting means saying no to spending, out loud, on camera.", "cite": 1},
        {"quote": "The trend reframes saying no as a flex.", "cite": 2}]
    assert set(re.findall(r"https?://[^\s\"']+", str(b))) <= {C1.url, C2.url}
    for k in ("format_idea", "hook", "timing_tip", "content_tip"):
        assert "viral" not in b[k].lower()

def test_no_invented_numbers():
    t = topic([C1]); b = br.build_brief(t, {})
    assert set(re.findall(r"\d+", b["format_idea"] + b["hook"])) <= set(re.findall(r"\d+", t.title))

def test_confidence_never_high():
    assert br.build_brief(topic([C1]), {})["confidence"] == "low"
    assert br.build_brief(topic([C1]), {})["confidence_basis"] == "1 source"
    assert br.build_brief(topic([C1, C2, C1]), {})["confidence"] == "medium-high"
```
Impl: §4.5. It reuses `recs.format_for`, `recs.hooks_for`, `recs.timing_for`, and `rk.host_of` (DRY).

### T16: scan rejects a bad niche with 400, before the key check [api]
```python
# tests/test_api.py
import httpx, pytest
from signalfire import you_client as yc, ranking as rk
from tests.conftest import SENTINEL, NICHE, load

@pytest.mark.parametrize("body", [{"niche": ""}, {"niche": "   "}, {}, {"niche": "x" * 81}])
def test_scan_invalid_niche_400(client, nokey, body):
    r = client.post("/api/scan", json=body)
    assert r.status_code == 400 and r.json()["error"] == "invalid_niche"
```
Expected fail: `ImportError: cannot import name 'create_app'`, then 422 ≠ 400. Impl: `ScanIn(niche: str | None = None)`, validated manually, returning `JSONResponse(400, …)`.

### T17: missing key on the live path → 503, generic body
```python
def test_scan_missing_key_503(client, nokey):
    r = client.post("/api/scan", json={"niche": NICHE})
    assert r.status_code == 503
    assert r.json() == {"error": "engine_unavailable", "message": "Live search is not configured."}
    assert "YOU_API_KEY" not in r.text
```
Impl: Call `require_key()` once before the fan-out, and map `EngineUnavailable` to the fixed body.

### T18: happy scan: 3 topics, recs, gource_log (Golden B)
```python
def test_scan_happy_golden(client, you):
    r = client.post("/api/scan", json={"niche": NICHE})
    assert r.status_code == 200, r.text
    j = r.json()
    assert you.calls.call_count == 3
    assert j["engine"] == {"provider": "you.com", "endpoints": ["search"], "queries": rk.queries_for(NICHE)}
    assert j["ranking"]["kind"] == "HEURISTIC" and j["generated_at"] == "2026-11-22T12:00:00Z"
    ts = j["topics"]
    assert len(ts) == 3
    assert [t["title"] for t in ts] == ["Loud Budgeting Is Taking Over Gen Z Money Talk",
        "Buy Now, Pay Later Use Spikes Ahead of Holidays", "The Cash Stuffing Comeback Nobody Expected"]
    assert [t["score"] for t in ts] == [7.0, 6.0, 4.4]
    assert [t["momentum"] for t in ts] == ["high", "medium", "medium"]
    for t in ts:
        assert t["score_kind"] == "HEURISTIC" and t["brief_available"] == (len(t["citations"]) > 0)
        for c in t["citations"]:
            assert set(c) == {"url", "title", "snippet"} and c["url"].startswith("http")
    rec = j["recommendations"]
    assert rec["available"] and rec["focus"]["topic_id"] == ts[0]["id"]
    assert rec["timing"]["window"] == "next 24h" and rec["hooks"] and rec["content_format"]["format"]
    log = j["gource_log"]
    assert len(log) == 17
    assert any("|meme|A|/personal-finance-for-gen-z/web/the-budget-cat-meme-is-everywhere" in l for l in log)
    assert any("|meme|D|/personal-finance-for-gen-z/web/the-budget-cat-meme-is-everywhere" in l for l in log)
    assert SENTINEL not in r.text and "will go viral" not in r.text.lower()
```
Impl: Use `ThreadPoolExecutor(3)` over `queries_for(niche)`. A slot that raised `UpstreamError` becomes `[]` and counts as a failure, and 3 failures → 502. Then `cands = rk.eligible(niche, slots, clock())`, `topics = rk.pick(niche, cands)`, `rec = recs.build(topics)`, and `log = gource.build_log(niche, cands, topics, rec, clock())`. Call `clock()` **once** per request and reuse the value.

### T19: partial failure OK; total failure 502; thin results 422
```python
def test_scan_partial_failure_ok(client, you):
    ok = httpx.Response(200, json=load("search_ok.json"))
    you.get(yc.SEARCH_URL).mock(side_effect=[ok, httpx.Response(500), ok])
    assert client.post("/api/scan", json={"niche": NICHE}).status_code == 200

def test_scan_all_fail_502(client, you):
    you.get(yc.SEARCH_URL).mock(return_value=httpx.Response(500, text=f"oops {SENTINEL}"))
    r = client.post("/api/scan", json={"niche": NICHE})
    assert r.status_code == 502 and r.json()["error"] == "upstream_error"
    assert SENTINEL not in r.text and "oops" not in r.text

def test_scan_insufficient_422(client, you):
    you.get(yc.SEARCH_URL).mock(return_value=httpx.Response(200, json={"results": {"web": []}}))
    r = client.post("/api/scan", json={"niche": "zzzz"})
    assert r.status_code == 422 and r.json() == {"error": "insufficient_signal",
        "message": "Not enough fresh sources for this niche. Try a broader niche.", "found": 0}
```
**[VERIFY]** respx returns the existing route when the same pattern is re-registered. If it doesn't, use `you.routes[0].side_effect = …`. Side-effect order is not tied to a specific query, which is fine because only success is asserted.

### T20: `/api/gource` returns the last scan's log (404 if none; failures don't overwrite)
```python
def test_gource_404_before_scan(client, nokey):
    r = client.post("/api/gource")
    assert r.status_code == 404 and r.json() == {"error": "no_scan", "message": "Run a scan first."}

def test_gource_returns_last_scan_log(client, you):
    scan = client.post("/api/scan", json={"niche": NICHE}).json()
    r = client.post("/api/gource")
    assert r.status_code == 200
    j = r.json()
    assert j["lines"] == scan["gource_log"] and j["format"] == "gource-custom-log"
    assert j["roles"] == ["search", "news", "meme", "rec"] and j["niche"] == NICHE

def test_failed_scan_does_not_overwrite_log(client, you):
    first = client.post("/api/scan", json={"niche": NICHE}).json()["gource_log"]
    you.get(yc.SEARCH_URL).mock(return_value=httpx.Response(200, json={"results": {"web": []}}))
    assert client.post("/api/scan", json={"niche": "zzzz"}).status_code == 422
    assert client.post("/api/gource").json()["lines"] == first
```
Impl: Set `app.state.last_gource = None` in `create_app`, and assign it only on a 200 scan.

### T21: brief for an unknown topic → 404
```python
def test_brief_unknown_404(client, nokey):
    r = client.post("/api/brief", json={"topic_id": "t_nope"})
    assert r.status_code == 404 and r.json()["error"] == "unknown_topic"
```

### T22: brief with zero citations → 409 refuse, no key, no HTTP (golden)
```python
def test_brief_zero_citations_409(client, nokey):
    client.app.state.topics["t_zero000000"] = rk.Topic(
        id="t_zero000000", title="Cash stuffing envelopes thread", score=1.0, momentum="low",
        breakdown={}, citations=[], is_news=False)
    r = client.post("/api/brief", json={"topic_id": "t_zero000000"})
    assert r.status_code == 409
    j = r.json()
    assert j["refused"] is True and j["error"] == "insufficient_citations" and j["citations"] == []
    assert "http" not in r.text
```
Impl: `app.state.topics = {}`. Order is 404 → 409 → `require_key()` → contents.

### T23: scan → brief → 200 with disclaimer and grounded tips (end-to-end)
```python
def test_scan_then_brief(client, you):
    topics = client.post("/api/scan", json={"niche": NICHE}).json()["topics"]
    t = topics[0]
    r = client.post("/api/brief", json={"topic_id": t["id"]})
    assert r.status_code == 200
    j = r.json()
    assert j["disclaimer"] == "SignalFire surfaces public signals. It does not guarantee virality. Trends are volatile; use your own judgment."
    assert j["engine"]["endpoints"] == ["contents"] and j["citations"] == t["citations"]
    assert j["hook"] and j["timing_tip"] and j["content_tip"].startswith("Put source [1]")
    assert all(1 <= w["cite"] <= len(j["citations"]) for w in j["why_now"])
    assert "viral" not in (j["hook"] + j["format_idea"] + j["timing_tip"] + j["content_tip"]).lower()
```

### T24: Contents down → snippets; missing key with cites → 503
```python
def test_brief_contents_down_falls_back(client, you):
    tid = client.post("/api/scan", json={"niche": NICHE}).json()["topics"][0]["id"]
    you.post(yc.CONTENTS_URL).mock(return_value=httpx.Response(500))
    r = client.post("/api/brief", json={"topic_id": tid})
    assert r.status_code == 200 and r.json()["why_now"]

def test_brief_missing_key_503(client, you, monkeypatch):
    tid = client.post("/api/scan", json={"niche": NICHE}).json()["topics"][0]["id"]
    monkeypatch.delenv("YOU_API_KEY")
    r = client.post("/api/brief", json={"topic_id": tid})
    assert r.status_code == 503 and r.json()["error"] == "engine_unavailable"
```
Impl: `try: text = contents(urls) except UpstreamError: text = {}`.

### T25: UI static checks [ui]
```python
# tests/test_ui.py
import re

def test_index_static_contract(client):
    r = client.get("/")
    assert r.status_code == 200
    h = r.text
    assert "SignalFire" in h and "HEURISTIC" in h
    for el in ('id="niche"', 'id="scan"', 'id="timer"', 'id="status"', 'id="topics"', 'id="brief"',
               'id="recs"', 'id="rec-focus"', 'id="rec-hook"', 'id="rec-timing"', 'id="rec-content"',
               'id="gource-tree"', 'id="gource-replay"', 'id="gource-ticker"', 'id="gource-empty"'):
        assert el in h, el
    for label in ("Focus", "Hook", "Timing", "Content"):
        assert f">{label}<" in h
    assert "/api/scan" in h and "/api/brief" in h and "/api/gource" in h
    for banned in ("innerHTML", "outerHTML", "insertAdjacentHTML", "document.write", "eval(", "new Function"):
        assert banned not in h, banned
    assert not re.search(r"""<(script|link|img|iframe)[^>]+(src|href)=["']?(https?:)?//""", h, re.I)
    assert "@import" not in h and "fonts.googleapis" not in h
    assert not re.search(r"""fetch\(\s*["'`](?!/api/)""", h)
    assert 'http-equiv="Content-Security-Policy"' in h
    assert "function safeHref" in h
    assert "Not git history" in h
    assert "will go viral" not in h.lower()
    assert "react" not in h.lower()
```
Expected fail: 404 or missing ids. Impl: §7. After it's green, check it manually in a browser with the live key. That's the only manual test.

### T26 (P1): blocklist unit test
```python
def test_blocklist_drops_titles():
    hs = [Hit(f"https://e.example/{i}", t, "s", None, "web") for i, t in
          enumerate(["NSFW money memes", "Alpha Bravo", "Charlie Delta", "Echo Foxtrot"])]
    assert all("nsfw" not in t.title.lower() for t in rk.rank("x", [hs], NOW))
```

### T27 (P1): live smoke [LIVE], off by default
```python
# tests/test_live_smoke.py
import os, pytest
from fastapi.testclient import TestClient
pytestmark = [pytest.mark.live, pytest.mark.skipif(
    os.environ.get("RUN_LIVE") != "1" or not os.environ.get("YOU_API_KEY"), reason="live only")]

def test_live_scan_gource_brief():
    from signalfire.app import create_app
    c = TestClient(create_app())
    r = c.post("/api/scan", json={"niche": "personal finance for Gen Z"})
    assert r.status_code in (200, 422), r.text
    if r.status_code == 200:
        j = r.json()
        assert len(j["topics"]) == 3 and j["gource_log"]
        assert c.post("/api/gource").json()["lines"] == j["gource_log"]
```
Run: `RUN_LIVE=1 python -m pytest -q tests/test_live_smoke.py`. Export the key in your own shell beforehand and never paste it into a file. This costs about 3 searches. Use the first run to settle every **[VERIFY]** item (URLs, header, params, response shape, freshness, the related-token threshold) and update fixtures and parsers together.

**Done means:** `python -m pytest -q` is fully green (live test skipped), one manual live scan → recs → tree → brief works in the browser at `http://127.0.0.1:8791`, and the §10 checklist passes.

---

## 6. Golden behavior (each one pinned by a test)

| # | Behavior | Tests |
|---|---|---|
| G1 | `POST /api/scan {niche}` → 200 with **exactly 3** topics, each with `title`, `score` + `score_kind:"HEURISTIC"`, and `citations[{url,title,snippet}]` | T9, T18 |
| G2 | The scan also returns `recommendations` with `focus`, `hooks[]`, `timing`, and `content_format`, derived only from cited topics. All URLs ⊆ topic citations. With no cited topic it returns the structured `available:false` refusal. | T10, T11, T18 |
| G3 | The scan also returns `gource_log`: lines `unix\|role\|type\|/synthetic/path`, ascending by unix. Role ∈ {search, news, meme, rec}, type ∈ {A, M, D}. Paths start with `/` and contain no URLs, IPs, ports, emails, hosts, or blocked titles. | T12, T13, T18 |
| G4 | `POST /api/gource` returns the last successful scan's log, or 404 `no_scan` if there is none. Failed scans don't overwrite it. | T20 |
| G5 | `POST /api/brief` for a topic with zero citations → **409** structured refuse, with no URL anywhere in the body and no key or engine call | T14, T22 |
| G6 | A brief with cites → 200 with the exact disclaimer footer **plus** `hook`, `timing_tip`, and `content_tip` grounded in the cites. Only citation URLs appear, and there are no invented digits. | T15, T23 |
| G7 | Ranking is a hand-set **HEURISTIC** (0.40 / 0.30 / 0.20 / 0.10), labeled in the API and UI. No science claim, and "will go viral" never appears. | T7, T11, T15, T18, T25 |
| G8 | Empty, whitespace, missing, or >80-char niche → **400**, checked before the key | T16 |
| G9 | Missing `YOU_API_KEY` on the live path → **503** generic body. No secrets, env names, or upstream bodies in any response. | T1, T3, T17, T19, T24 |
| G10 | Missing data is handled: partial failure still works, all-fail → 502, thin → 422 (never padded), Contents down → snippets | T2, T19, T24 |

---

## 7. UI spec (Design /5) [CONTRACT for ids and labels; DECISION for look]

One file, `signalfire/static/index.html`, with inline `<style>` and `<script>`. No React, CDN, webfonts, images, or build step.

**Head:** `<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self'; base-uri 'none'; form-action 'none'">`. `<title>SignalFire · Trend Radar</title>`.

**Look (dark editorial):**
- Background `#0E0F12`, surface `#16181D`, hairline `#2A2D34`, text `#ECE8E1`, muted `#9A968E`, accent (signal fire) `#FF5A36`.
- Role colors: search `#7AA2FF`, news `#FFB86B`, meme `#FF6AD5`, rec `#5AF2A0`.
- Headlines use the system serif stack (`"Iowan Old Style", Georgia, serif`), large and tight. Body and UI use the system sans stack (`-apple-system, "Segoe UI", system-ui, sans-serif`). Numbers use `font-variant-numeric: tabular-nums`.
- Layout: a masthead (wordmark "SignalFire" plus the line "The next topic, with sources. Or we refuse."). Below it, a big niche input with the **Scan** button and the `#timer`. Then a two-column grid at ≥ 1100 px: the left column (≈ 62%) holds `#topics` and `#brief`, and the right column (sticky) holds `#recs` and the tree. Below 1100 px it stacks in the order input → topics → recs → tree → brief.

**Elements and behavior:**
- `#niche` (placeholder `personal finance for Gen Z`, `maxlength=80`), `#scan` button, and `#timer`, which ticks elapsed seconds while scanning. At 15 s, `#status` shows "Still searching the live web…" and keeps waiting. Enter submits.
- `#status` (`role="status"`, `aria-live="polite"`) shows the server's `message` for 400/422/502/503/409, styled as a visible red-accent banner. It never shows raw JSON. 503 shows: "Live search is not configured. Set YOU_API_KEY in the server's environment."
- **`#topics`: 3 cards.** Each shows the rank numeral (large serif), the title, a role chip (color by role), and the score pill `HEURISTIC 7.0/10 · high momentum`. A `<details>` "Why this score" lists the breakdown and weights. **Source chips are always visible**: one per citation, with the bold domain, the title as a link (`target="_blank" rel="noopener noreferrer"`, href only through `safeHref`), and the snippet in muted text. A **Write brief** button is always clickable so the 409 is visible. When `brief_available:false`, the card carries the badge "No usable sources".
- **`#recs`: Recs panel** with four labeled blocks, `#rec-focus` (`<h3>Focus</h3>`), `#rec-hook` (`<h3>Hook</h3>`), `#rec-timing` (`<h3>Timing</h3>`), and `#rec-content` (`<h3>Content</h3>`). Each block names the topic it's tied to ("from #1 · Loud Budgeting…"), and hooks render as a list with a source chip. `recs.note` is shown in small text. When `available:false`, the panel shows `message` in place of the 4 blocks.
- **Tree:** an `<svg id="gource-tree" viewBox="0 0 600 600" role="img" aria-label="Trend tree of this scan">` with a caption reading "Gource-format log of this scan. Synthetic paths built from live You.com results. Not git history."
  - After a 200 scan, the UI calls `POST /api/gource` and plays `lines`, one event every 220 ms (playback goes by event order, not real time).
  - Layout (deterministic, no physics): the niche root sits at center (300,300). Platform branches `web`, `news`, `social`, and `rec` sit at fixed angles (−90°, 0°, 90°, 180°) at radius 120. A branch appears on its first event. Leaves fan out at radius 240 within ±40° of their branch angle, evenly spaced by arrival order, and existing leaves re-space as new ones arrive.
  - **A** adds a leaf circle (r = 6, role color) with an edge to its branch and a `<text>` label (slug, truncated to 18 chars). **M** grows the leaf to r = 10 and adds an accent ring. **D** fades the leaf and edge to opacity 0.15. **rec** leaves use the rec color.
  - `#gource-ticker` shows the current event as `YYYY-MM-DD HH:MM UTC · role · A/M/D · platform/slug`. There's a role legend, and `#gource-replay` replays the log.
  - `#gource-empty` reads "No scan yet. The tree plays after a scan." and is shown when there are no lines, when `/api/gource` returns 404, or after a failed scan (the tree is cleared locally).
  - P1: `#gource-download` builds a `Blob` of `lines.join("\n")` and downloads `signalfire-scan.gource.log` so anyone can open it in real Gource offline.
- **`#brief`:** shows the format idea, hook, timing tip, content tip, why-now quotes with `[n]` anchors to the citation list, confidence with its basis, and citations. **The disclaimer footer is always rendered.** A **Copy brief** button copies client-built plain text via `navigator.clipboard.writeText`. A refusal (409) renders a distinct panel: "Refused: no usable sources. SignalFire won't write a brief without citations." with no links.
- A footer under the results reads: "Scores are a hand-set HEURISTIC (recency, corroboration, cross-query hits, news). Not a prediction. Never 'will go viral'."

**Security [CONTRACT]:** All dynamic text goes through `textContent`, `createElement`, or `createElementNS` (SVG). `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, and `eval` are never used. `function safeHref(u){ return /^https?:\/\//i.test(u) ? u : "#"; }` guards every anchor. `fetch` goes only to relative `/api/...` paths. Source titles and snippets are untrusted web text.

---

## 8. Judge mapping (each /5)

| Criterion | What judges see | Why it scores |
|---|---|---|
| **Design**: would a creator enjoy it? Sources visible? | A dark editorial page with one big input and a ticking timer. Three cards with **always-visible source chips**. A four-block recs panel that answers "what do I make first". A tree that blooms as the scan replays, with ranked leaves glowing and dropped ones fading. A copy-ready brief with the disclaimer. | It feels like a creator tool, not a form. Sources are visible without a click. The tree is the memorable beat. |
| **Uniqueness**: only possible with live-web AI? | Every topic, score input, hook quote, timing window, and tree leaf comes from this week's You.com results. Pull the key and you get a 503, with no output. The cite-or-refuse brief and recs refusal are the differentiators. | It can't work on a model trained last year, and it refuses rather than hallucinating. |
| **Technical Execution**: works live? Handles missing data? | A live scan on stage. Visible failure states: "No usable sources" badge, 409 refusal panel, recs refusal, empty tree, 422 "try broader", 502 retry, 503 key. Green pytest run in the backup video. | Missing-data paths are designed in and pinned by tests (G4, G5, G8–G10). |
| **Business Execution**: customer, market, who pays? | The demo ends on the buyer: the mid-tier creator or solo social manager already paying for research they don't trust. SignalFire sells that hour back. | Named user, named pain, named payer. |
| **Use of You.com**: engine, not garnish; name endpoints | The response shows `"engine":{"provider":"you.com","endpoints":["search"]}` with the 3 queries (trend/news/meme), and the brief shows `["contents"]`. Said out loud: "Search finds, ranks, and recommends. Contents grounds the brief." | You.com is the only data source. No You.com means no product. |

---

## 9. Demo script (≈ 80 s; Frank speaks, one driver at the keyboard)

Rehearse it twice. Record a backup video (1–3 min, which doubles as the Devpost video) **by 15:30**.

- **0:00–0:08, the problem, not the stack.** "Creators lose an hour a day hunting what's next, and half of what they find, they can't source. Watch."
- **0:08–0:20, live scan.** Type `personal finance for Gen Z`, hit **Scan**, and point at the timer. "That's You.com Search: trend, news, and meme queries against this week's web. Live."
- **0:20–0:35, cards.** "Three topics, ranked. The score says HEURISTIC right on it: freshness, how many sources agree, news coverage. Not a crystal ball. Every card shows its sources. Click one; they're real."
- **0:35–0:48, recs.** Point at the recs panel. "What to make first. Focus: this one. Hooks, and that one's a quote straight from the source. Timing: post within 24 hours, because the lead source is under a day old. Format: short explainer."
- **0:48–0:58, the tree.** Hit replay. "This is the scan as a Gource log. Every leaf is a source You.com found, timed to when it was published. Ranked ones glow, dropped ones fade. Synthetic paths, real results. Not git history."
- **0:58–1:08, brief and refuse.** Click **Write brief** on #1. "Hook, timing, content tip, each tied to a numbered source. Footer: no virality guarantee." Then click a "No usable sources" card (or cut to the 5 s refusal clip from the backup if today's scan has none). "And when there are no sources, it refuses. No fake links. No fake stats."
- **1:08–1:20, the close. Frank says it, then stops talking:**
  > "Creators don't need another posting bot. They need the next topic, with sources, or we refuse to write it. That's SignalFire. You.com is the live web doing the work. The buyer is the creator already paying for research they don't trust. We sell the hour back. **That's who writes the check.**"

Each clause of the close is literally true in the product. *Not a posting bot*: there's no posting code (§10). *Next topic, with sources*: G1, G2. *Or we refuse*: G5 and the recs refusal. *You.com doing the work*: §3, G9. *Who pays*: §8 Business. End there. No "so yeah."

---

## 10. DQ checklist (run at 15:30, before the Devpost submit)

| Risk | Check | Pass condition |
|---|---|---|
| **Paper-only** | A live `http://127.0.0.1:8791` scan, recs, tree, and brief work on stage. `inputs/research.md` is a leave-behind only. | The demo is the running app |
| **Slides-as-entry** | The Devpost video is the app in use, and slides are optional context | ≥ 80% of the video is screen capture of the app |
| **Frozen chatbot wrapper** | There's no LLM call anywhere, and every output traces to You.com (T11/T15 URL subsets, T17 503 without key) | `grep -rniE "openai\|anthropic\|chat/completions\|llm" signalfire/` → nothing |
| **Posting bot** | No posting, scheduling, or social-auth code | `grep -rniE "oauth\|schedule\|tweet\|/post\b" signalfire/` → nothing relevant |
| **TikTok scrape** | No TikTok calls, scraping, or fake TikTok data. A tiktok.com URL in You.com results is fine. | `grep -rni "tiktok" signalfire/` → only `SOCIAL_HOSTS` in `ranking.py` |
| **Secrets in git** | The key comes only from the env. `.gitignore` covers `.env*`. No key strings, no `.env` files. | `git grep -nI "YOU_API_KEY="` → nothing; `git grep -nIE "[A-Za-z0-9_-]{32,}" -- signalfire tests` → no real keys (the sentinel is fine); `git log -p \| grep -iE "x-api-key: ?[a-z0-9]{8,}"` → nothing; `ls -a \| grep -E "^\.env"` → nothing |
| **Gource-of-git-history pretending to be live web** | The log is built only from the current scan's You.com results (T13 golden, T20 last-scan). There's no git, subprocess, or repo walk. The UI caption says "Not git history." | `grep -rnE "subprocess\|os\.system\|git log\|\.git/" signalfire/` → nothing; the T25 caption assert passes; a second niche produces a different log |
| **Public GitHub** | The repo is public, and the README has run/test instructions and the §1 Devpost text | Opens in an incognito browser |
| **Devpost fields** | Name, one-liner, problem/creators/what it does, **Track: Trend Radar**, You.com endpoints (Search, Contents) and what they power, tools list, 1–3 min video | All filled before 16:00 |
| **No "viral" claims** | `grep -rni "will go viral" signalfire/` → only the UI footer's "Never 'will go viral'" line | The UI shows momentum and confidence labels only |

---

## 11. Out of scope and honest gaps

**Out of scope today (do not build):** posting or scheduling, calendars, TikTok (API or scrape), YouTube (optional per PRODUCT.md; skipped so the demo runs on You.com alone), training, calibration, Precision@K or any eval harness, embeddings or clustering, a safety classifier beyond the 5-word blocklist, any LLM brief writer, React or any build step, a database or Redis, auth or accounts, share-to-X, deployment or Cloudflare, Hermes ports, the You.com Research and Answer endpoints, the Gource OpenGL binary or video rendering, and running anything against git history.

**Honest gaps:**
- **[GAP] The heuristic is unvalidated.** The weights 0.40 / 0.30 / 0.20 / 0.10 are hand-set, and scores are relative within one scan. "Predictive" means forward-looking rules over early signals. We make no accuracy claim, and `research.md`'s formula and metrics are aspirational.
- **[GAP] Recommendations are rules, not judgment.** Focus is the top cited topic, hooks are templates plus one verbatim snippet, timing is a recency bucket, and format is a 4-way rule. They're plain by design, trading flair for zero invented claims.
- **[GAP] The "meme" role is query membership.** A result counts as a meme signal because You.com returned it for `"{niche} meme"`, not because we classified it. In practice a meme can rank low (Golden B: M1 shows in the tree but isn't picked).
- **[GAP] A "topic" is a headline.** Topics are real article titles grouped by a crude shared-token rule, and near-duplicates are possible when the fill relaxes distinctness.
- **[GAP] The tree is synthetic.** Paths are slugs of niche/platform/title, and timestamps are source `page_age` clamped to 7 days. It replays one scan and is not a live crawl. Slugging drops dotted tokens (for example "2.0"), which is an accepted loss.
- **[GAP] Recency depends on `page_age`.** Undated results score 0 on recency, get "Timing unclear", and appear at scan time in the tree. You.com freshness semantics are **[VERIFY]**.
- **[GAP] Verbatim snippets can contain any words.** Why-now quotes and the fallback snippet are attributed source text with `[n]`. Hooks and quote selection skip text containing "viral", but a fallback snippet is shown as-is, as a cited quote and never as SignalFire's claim.
- **[GAP] A live refusal can't be forced.** Whether today's scan yields a zero-citation topic depends on the web. The demo shows it live if it happens, and otherwise uses the recorded clip plus T22. We never fake one.
- **[GAP] Session memory only.** A restart loses topics and the last log (404s). The app is single-process, localhost-bound, and unrate-limited.
- **[GAP] You.com details are unverified offline.** URLs, the auth header, params, and response fields are marked **[VERIFY]**, and T27's first live run settles them.
- **[GAP] US/English only.** It uses `country=US` and English stopwords.
- **[GAP] Cost.** A scan is 3 searches and a brief is 1 Contents call. At $10 ≈ 2,000 searches that's roughly 600 scans. Never put scans in a loop.
- **[LIVE] Latency.** The 15 s target assumes You.com responds in under 8 s per call. Slower responses show "still searching", and a timeout produces a visible 502, never a silent hang.

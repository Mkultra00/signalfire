# SignalFire — product cut (hackathon MVP)

One job: **niche in → 3 ranked viral-trend / meme topics with You.com cites → recommendations (focus, hook, timing, content) → a Gource-format trend tree → a brief that refuses if cites are missing.**

Not a posting bot. Not a calendar. Not TikTok scrape. Not an eval harness. Not a paper.

## User
Mid-tier independent creator or solo social manager planning 1–3 days ahead.

## Happy path
1. Type a niche (example: `personal finance for Gen Z`).
2. Scan (~15s target; fail visibly if slower / error).
3. See **exactly 3** ranked topics (trends + memes). Each row: title, heuristic momentum, **visible source URLs** from You.com.
4. See **recommendations** for what to make first: focus topic, hook options, timing window, content format. Labels only — never “this will go viral.”
5. See a **Gource-format trend visualizer**: custom event log of the scan (`unix|role|A|M|D|/niche/platform/topic`) played as an in-page tree. Roles = signal sources (`search`, `news`, `meme`, `rec`). Paths synthetic. Not Gource OpenGL. Not a live hunt.
6. Click one topic → Creator Brief: format idea, hook, timing tip, content tip, confidence label, citations, mandatory disclaimer.
7. If You.com returns no usable cites for that topic: **refuse to write the brief**. No invented URLs. No fake stats.

## UI (Design /5)
Must look like a creator tool, not a form. One `index.html`, no React, no CDN.
- Dark editorial canvas, large niche input, three topic cards with source chips.
- Right or lower pane: Gource tree of the last scan (nodes appear as events play).
- Recs panel: Focus / Hook / Timing / Content — four labeled blocks, each tied to a cited topic.
- Missing data is visible (empty tree, 409 brief, 503 key). Fail loud, not silent.

## Honest limits (must appear in TDD)
- Ranking and “predictions” are a **heuristic**, not measured science. Do not claim a trained model, Precision@K, or guaranteed virality.
- TikTok Research API is out. Do not scrape. Do not fake live TikTok.
- YouTube is optional garnish if a key exists; **You.com is required**. Demo must still work with You.com alone.
- Research PDF in `inputs/research.md` is **spec / leave-behind**, not an experiment.
- Never display “this will go viral.” Momentum / confidence labels only.
- Gource is a **custom log + in-page tree** of this scan. Not git history. Not swarm-attack viz. No real URLs/IPs in log paths.
- Mandatory brief footer: “SignalFire surfaces public signals. It does not guarantee virality. Trends are volatile; use your own judgment.”

## Stack (YAGNI for remaining hours)
- Python 3.10+ FastAPI backend + one `index.html` (no React).
- Bind demo to `127.0.0.1` on a **new** port. Never 8080, 8766, 8768. Prefer **8791**.
- You.com client module with mock in tests.
- Gource log builder in Python; UI reads `/api/gource` after scan.
- No Redis/Postgres. Session-memory is fine.
- Tests: pytest. TDD: write failing tests first in the later implement pass — this document specifies those tests now.

## Out of scope today
Posting, scheduling, TikTok, training, calibration, embeddings, safety classifier beyond a tiny keyword blocklist, Cloudflare, Hermes ports, keys in git, Gource OpenGL binary, React, LLM brief writer.

## Who pays (business box)
The creator already paying for research they don’t trust. SignalFire sells the hour back.

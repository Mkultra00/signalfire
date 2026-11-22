TDD ONLY. Do not implement. Do not create product code (no app.py, server.py, index.html, tests/*.py, pyproject, package.json). Do not edit inputs/. Do not start servers. Do not touch ports 8080, 8766, 8768. Do not invent API keys or write .env files.

Read these files (relative paths, cwd is the project root):
- inputs/REQUIREMENTS.md
- inputs/PRODUCT.md
- inputs/event-packet.md
- inputs/research.md  (background spec; PRODUCT.md wins on any conflict)
- docs/TDD-signalfire-v1.md  (prior TDD — reuse contracts that still fit; supersede where PRODUCT.md now adds UI, recs, Gource)

Write the full replacement TDD to: docs/TDD-signalfire.md

The TDD must make a later implementer (Claude Code, second pass) able to ship without guessing.

Required sections:
1. Goal (one sentence) + Devpost track (Trend Radar) + one-line description for the form. Product now: predictive trend/meme analyzer + recs (focus, hooks, timing, content) + Gource-format trend tree + cite-or-refuse brief. You.com is the engine.
2. Architecture (2–3 sentences) + file list expected on implement (exact paths). Include gource log module + UI tree. No React. No LLM. Port 127.0.0.1:8791.
3. You.com engine: Search + Contents (prefer; [VERIFY] URLs). What they power (scan, cites, recs evidence). Missing data: 409 brief, 503 no key. Tests mock HTTP. Live key via env name YOU_API_KEY only — never a value.
4. Contracts with labels [CONTRACT] / [DECISION] / [LIVE] / [GAP] / [VERIFY].
5. Bite-sized TDD tasks (each: failing test snippet, command, expected fail, minimal impl note). Tests first. pytest. Cover: scan, ranking HEURISTIC, recs fields, gource custom-log lines, brief 409, UI static checks if any.
6. Golden behavior:
   - POST scan(niche) → exactly 3 topics, each with title, heuristic score, citations[{url,title,snippet}].
   - Scan also returns recommendations: focus, hooks[], timing, content_format — derived from cited topics, no invented sources.
   - Scan also returns gource_log: lines `unix|role|type|/synthetic/path` sorted ascending. Roles in {search,news,meme,rec}. Paths start with /. No URLs, IPs, ports, emails in paths.
   - POST /api/gource returns the last scan’s log (404 if none).
   - POST brief(topic_id) with zero citations → structured refuse, HTTP 409, no invented sources.
   - Brief with cites includes disclaimer footer PLUS hook + timing + content tips grounded in cites.
   - Ranking formula labeled HEURISTIC; weights documented; do not claim science. Never say “will go viral.”
   - Empty niche → 400. Missing YOU_API_KEY on live path → 503 with generic error, no secrets in body.
7. UI spec (Design /5): dark editorial single page; niche input; 3 topic cards with source chips; recs panel (Focus/Hook/Timing/Content); in-page Gource tree that plays the custom log. Fail states visible. No CDN, no innerHTML XSS. Vanilla JS.
8. Judge mapping: Design / Uniqueness / Execution / Business / You.com — how the demo hits each /5.
9. Demo script (60–90s) ending on who pays. Last line must enable Frank’s bang-close from REQUIREMENTS.md.
10. DQ checklist: paper-only, slides-as-entry, frozen chatbot wrapper, posting bot, TikTok scrape, secrets in git, Gource-of-git-history pretending to be live web.
11. Out of scope + honest gaps.

YAGNI. DRY. No React. No TikTok. No eval harness. No Gource OpenGL. No LLM.

When the file is written, stop. Do not implement.

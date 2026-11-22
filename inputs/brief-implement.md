# Implement SignalFire from the TDD (go build)

Read `docs/TDD-signalfire.md` (v2, 1212 lines). That doc is law. `inputs/PRODUCT.md` wins vs `inputs/research.md` on conflict. Do **not** edit the TDD, `inputs/`, or this brief.

## Output
- Implement the exact file tree in TDD §2.
- Tests **first**: T0–T25 are P0. Then T26–T27.
- Append a running log to `BUILD_LOG.md` (what went green, what you [VERIFY]-updated, what you could not verify).
- Do **not** `git commit` or `git push`. Kov commits.
- Do **not** write `.env`, `.env.example`, or any key value. Env name only: `YOU_API_KEY`.
- Do **not** start Hermes or touch ports 8080, 8766, 8768.
- Product bind: loopback port `8791` only (if 8791 busy, 8792 and note it in README). Do not bind all interfaces.
- No LLM. No mock/demo mode in product code. No Gource binary. No React. No CDN.
- README is the judges' project page: challenge, how to run, You.com Search+Contents, honest limits from §11, layout, MIT. Write `localhost:8791` — never a dotted IPv4 in README/text.
- If live You.com docs disagree with a **[VERIFY]** URL/header/param/shape: update fixture + parser together in one place and note it in `BUILD_LOG.md`. Tests mock HTTP. Do not invent an API key. Do not run `RUN_LIVE=1` unless a key is already in the process env (it probably is not — skip T27).

## Loop
For each task: write the test → run it → see it fail for the stated reason → minimal impl → green → full suite.

Done when `python -m pytest -q` is fully green (live test skipped).

Honesty: flag anything unverified. Never fabricate You.com payloads as if live.

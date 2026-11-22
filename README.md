# SignalFire

Creator Economy Hack (You.com), NYC, 2026-11-22. Track: **Trend Radar**.

Type a niche. Get 3 rising trends/memes from today's web with You.com sources, what to make first (focus, hook, timing, format), a Gource-format trend tree of the scan, and a brief that **refuses** to write itself without citations.

You.com **Search** is the engine (trend / news / meme queries). You.com **Contents** grounds the brief. No You.com result means no topic. No usable citation means no recommendation and no brief. Scores are a hand-set **HEURISTIC**, not a prediction model.

## Run

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
export YOU_API_KEY=   # your key; never commit it
uvicorn signalfire.app:app --host localhost --port 8791
```

Open `http://localhost:8791`

Tests: `python -m pytest -q`

## Honest limits

- Heuristic weights are unvalidated.
- Recommendations are rules over cited sources, not a strategist.
- "Meme" means the result came back on the `{niche} meme` query.
- Localhost demo, session memory only, US/English.

Spec: `docs/TDD-signalfire.md`. MIT.

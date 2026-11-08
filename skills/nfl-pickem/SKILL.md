---
name: nfl-pickem
description: Weekly NFL winner picks from the personal "NFL Pick'em" engine — beat-the-favorite via a 3-expert consensus panel + agentic judge (default), with strong-underdog and Monte-Carlo SA as fallbacks. Use when the user asks to run the NFL pickem, get/update this week's NFL picks, pick the slate, compare consensus vs strong_underdog, change the default strategy, or backtest the NFL pick engine.
---

# NFL Pick'em

Weekly NFL winner picks, backed by the `projects/nfl-pickem/` engine (its own git repo, 170+ tests).
Lord Bryson's standing requirement: **every pick carries a 1–16 confidence** and output is a plain-text brief by default.

**Current default strategy: `consensus`** (3 distinct expert models → agentic judge), set by Lord Bryson 2026-11-08. `strong_underdog` and `engine` remain available via `--strategy`.

## Where the code lives

- **Engine:** `/home/sidney/.openclaw/workspace/projects/nfl-pickem/` (run from this dir)
- **Status doc (single source of truth):** `/home/sidney/.openclaw/workspace/projects/NFL_PICKEM_STATUS.md`
- **Data cache:** `projects/nfl-pickem/data/` (re-fetchable ESPN JSON; git-ignored)
- **Audit trail:** `projects/nfl-pickem/data/consensus_log/` (one JSON per LLM panel decision — safe to resume after a crash)

## Run the current week

```bash
cd /home/sidney/.openclaw/workspace/projects/nfl-pickem

# Full week (Thu+Sun+Mon merged) with the default strategy (consensus) + plain-text brief
python3 pickem.py --slate --present

# Explicit strategy
python3 pickem.py --slate --present --strategy strong_underdog   # deterministic rule
python3 pickem.py --slate --present --strategy engine            # legacy SA + H2H
python3 pickem.py --slate --json                                  # machine-readable

# Anchor a specific week
python3 pickem.py --slate --today 20261108 --present
python3 pickem.py --week 20261101 --strategy consensus --present
```

Notes:
- `--present` is the **default** display; `--table` opts into the classic columnar table.
- `--offline` = cache-only (no network). Omit it to refresh stale cache.
- Consensus is slower (3 experts + judge per game, ~30–40s/game on the Ollama boxes at
  `192.168.1.6` / `192.168.1.8`). A 15-game slate takes ~10 min. Run it in the background
  and poll; per-game audit JSONs land in `data/consensus_log/` as they complete.
- If a run is interrupted, **do not re-run all 15** — read the fresh `data/consensus_log/*.json`
  (mtime after the run start) and only vote the missing games, then assemble the brief from the
  audit files. This is the resume path.

## Reporting to the user (conventions)

- Lead with the **plain-text brief**: each pick boxed on the left, `/16` confidence, and `[FLIP]`
  markers for any pick that goes against the spread favorite.
- **Delta-first:** when a strategy or data change affects already-submitted picks, report only
  the picks that changed (and how), unless asked for the full slate.
- Be honest about calibration: consensus is often more home-field-weighted than `strong_underdog`;
  `strong_underdog` is the documented rule that beats the favorite in backtests. Never overclaim
  "the panel is smarter" — surface the tradeoff.
- Keep the in-service command and any default-strategy change recorded in `NFL_PICKEM_STATUS.md`.

## Changing the default strategy

The default lives in one place: `pickem.py` → `ap.add_argument("--strategy", ..., default=...)`.
Flip it, re-run the full test suite (`python3 -m pytest -q` in the project dir), update the
"DECISION" block in `NFL_PICKEM_STATUS.md`, and `git commit` the decision in `projects/nfl-pickem/`.

## Backtests / research (context)

- **Leg B (2024+2025, 528 games):** consensus 65.7% (CI [61.7,69.8]) vs record-favorite 64.2%
  vs always-home 53.2%. See `reports/05_consensus.md`.
- **Odds research:** `reports/06_historical_odds_research.md` (free nflverse PFR spreads is the
  no-cost path to a true beat-favorite backtest; ESPN has no historical odds).
- Reports index: `projects/nfl-pickem/reports/`.

## Key files

- `pickem.py` — CLI (default display, strategies, slate/week, JSON).
- `strong_underdog.py` — the deterministic beat-the-favorite rule + `*_confidence` helpers.
- `consensus/{experts,judge,pipeline}.py` — 3-expert panel + agentic judge.
- `confidence.py` — single source for the 1–16 confidence mapping.
- `present.py` — plain-text weekly brief.
- `adapters/{espn,build_week}.py` — keyless ESPN fetchers (cache-first) + feature-card builder.
- `backtest_legb.py` — Leg B backtest harness. `tests/` — full suite.

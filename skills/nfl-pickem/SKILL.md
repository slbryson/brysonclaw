---
name: nfl-pickem
description: Weekly NFL winner picks from the personal "NFL Pick'em" engine — beat-the-favorite via a 3-expert consensus panel + agentic judge (default), with strong-underdog and Monte-Carlo SA as fallbacks. Tracks historical pick accuracy (Week 8+), tokens per week, and per-algorithm KPIs in a local ledger + dependency-free SVG/HTML dashboard. Use when the user asks to run the NFL pickem, get/update this week's NFL picks, pick the slate, compare consensus vs strong_underdog, change the default strategy, backtest the NFL pick engine, or view accuracy KPIs / the accuracy dashboard.
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
- **Accuracy ledger (R10):** `projects/nfl-pickem/data/history.json` — committed, single source of truth for historical picks + accuracy + tokens (Week 8+)
- **Accuracy dashboard (R10):** `projects/nfl-pickem/reports/accuracy_dashboard.html` — committed, opens in any browser, no JS/deps

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

- `pickem.py` — CLI (default display, strategies, slate/week, JSON, `--record`/`--seed-ledger`/`--render`).
- `strong_underdog.py` — the deterministic beat-the-favorite rule + `*_confidence` helpers.
- `consensus/{experts,judge,pipeline}.py` — 3-expert panel + agentic judge + per-game token totals.
- `confidence.py` — single source for the 1–16 confidence mapping.
- `present.py` — plain-text weekly brief.
- `scoreboard.py` — R10 local accuracy ledger (`data/history.json`) + KPI engine (idempotent upsert, Brier, confidence, flips).
- `charts.py` — R10 dependency-free SVG/HTML charts + dashboard renderer (no matplotlib, no JS).
- `adapters/{espn,build_week}.py` — keyless ESPN fetchers (cache-first) + feature-card builder.
- `backtest_legb.py` — Leg B backtest harness. `tests/` — full suite (211 tests as of 2026-11-08).

## R10 — Historical accuracy KPI

Lord Bryson's standing requirement (2026-11-08): maintain a durable local ledger of every pick,
scored against actuals, starting with **Week 8**, and expose:

- **Accuracy week-over-week** per algorithm
- **Tokens spent per week** per algorithm (consensus spends; deterministic strategies report 0)
- **Accuracy per algorithm, week over week**
- **Brier score** (when probabilities are available)
- **Confidence (1–16)** per pick, averaged per week

All visible the moment you open the repo — `reports/accuracy_dashboard.html` is a committed
artifact (not something you have to regenerate).

**Quick commands:**

```bash
cd /home/sidney/.openclaw/workspace/projects/nfl-pickem

# Idempotently seed Week 8 into the ledger (does nothing if already present)
python3 pickem.py --seed-ledger

# After a scored week, record it (default-on for scored weeks when you run --slate)
python3 pickem.py --slate --record

# Regenerate the dashboard from the ledger (no network needed)
python3 pickem.py --render

# Override the week label / use an alternate ledger file
python3 pickem.py --slate --record --label W9
python3 pickem.py --record --ledger /tmp/alt_history.json
```

**Week 8 seed (verified against the audited Nov-3 reports):**
- consensus 9/12 (misses: ARI@DAL, KC@DEN, MIA@NE) → 75.0%
- strong_underdog 11/12 (miss: ARI@DAL) → 91.7%

**Token plumbing:** `LLMClient.chat()` reads Ollama's `prompt_eval_count` / `eval_count` payload
into `client.last_usage`; `call_expert` / `call_judge` copy it into `ExpertResponse.tokens` /
`JudgeResponse.tokens`; `ConsensusResult.tokens` is the sum across all experts + judge; the row in
`pickem.py` carries `tokens`; `record_run()` stores the sum into the ledger. Unit-tested in
`tests/test_scoreboard.py::test_call_expert_propagates_tokens`.

**Where the KPI lives in the repo:**
- `PROJECT_PLAN.md` — the full R10 contract (schema, week keying, token plumbing, DoD)
- `README.md` — the quick-reference commands + the KPI explanation
- `tests/test_scoreboard.py` + `tests/test_charts.py` — the unit tests

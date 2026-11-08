# NFL Pick'em — Project Status

**Last updated:** 2026-11-08 12:40 CST · **Owner:** Brysonclaw 👺 · **For:** Lord Bryson

## ✅ DECISION (2026-11-08 12:37 CST — Lord Bryson)

- **Consensus (3-expert panel + agentic judge) is now the DEFAULT strategy** in `pickem.py`
  (was `strong_underdog`). `strong_underdog` and `engine` remain available via `--strategy`.
- In-service command is now simply: `python pickem.py --slate [--present]`
- `strong_underdog` stays as the documented fallback path (still the rule used in Leg B backtests).

## ✅ Milestone 3: LEG B BACKTEST — FULL 2024 + 2025 (completed 2026-11-04 08:02 CST)

528 games, spread-agnostic (2024/25 have scores, no cached odds), leakage-free (point-in-time
records/PPG only), 0 hard-fails across 2,112 LLM calls, every game resolved by the judge.

| Method | Hits | Rate | 95% CI |
|---|---|---|---|
| **Consensus panel** | **347/528** | **65.7%** | **[61.7%, 69.8%]** |
| Prior-record favorite (stand-in) | 339/528 | 64.2% | [60.1%, 68.3%] |
| Always-home | 281/528 | 53.2% | [49.0%, 57.5%] |

- Season split: 2024 = 68.7%, 2025 = 62.7%.
- Head-to-head where panel and record-favorite disagree (52 games): panel right **30–22 (58%)**.
- **Verdict:** beats always-home decisively (CIs disjoint); modest, *earned* edge over the record-
  favorite (small and CI-overlapping, so not a "proven" dominance). Robustness + auditability are
  the real wins. Default stays `strong_underdog`; `consensus` = robust fallback path.
- Artifacts: `data/legb_results.jsonl` (528 rows), `data/legb_audit/` (all LLM I/O),
  `reports/05_consensus.md`. Rerun: `python backtest_legb.py --report`.

## ✅ Milestone 2: 3-EXPERT CONSENSUS + AGENTIC JUDGE (2026-11-03)

Mandate: build a 3-way expert consensus vote with an agentic judge that makes the final
pick; then backtest to the beginning of the season.

**Pipeline DONE — built, tested (148/148 green), and verified live.**
- 3 **genuinely distinct** expert models → agentic **judge** → deterministic fallback chain.
- Anti-leakage guard (raises if any final/score/winner field leaks into the pre-game card).
- Full JSON audit of every LLM call + decision. Wired: `--strategy consensus`.
- **First-week result (2026-11-01, n=12, 12/12 resolved by judge, 0 hard-fails):**
  - Consensus **9/12 (75%)**  <  Favorite **10/12 (83.3%)**  <  **Strong-underdog 11/12 (91.7%)**
  - **Honest: the LLM panel did NOT beat the rule on this week.** It's a less-calibrated version
    of the underdog heuristic (rightly caught the LV upset, wrongly flipped KC, unanimously
    missed the NE 7-point-underdog win). On n=12 this is within noise — no method is "proven."
- Report: `reports/05_consensus.md`. Commit `a3b544c`.

**Backtest-to-season: BLOCKED BY DATA (not code).**
- 2026 is synthetic: only `20261101` (12 g) + `20261102` (1 g) exist; every other 2026 date 404s.
- 2024 & 2025 have **final scores but NO historical spreads/odds** → "beat the favorite" is
  **not computable** for those seasons without a historical-odds source.
- **DECISION OPEN (Lord Bryson):** (a) allow a ~1,000-LLM-call spread-agnostic 2024/25 sweep
  (compares vs always-home, *not* favorite), (b) provide a historical-odds key for a true
  multi-season beat-favorite run, or (c) cap the sweep to a sample + confidence interval.

## ✅ Milestone 1: BEAT THE FAVORITE (2026-11-03)

Mandate: *"beat the always-pick-favorite baseline, iterate ≤3×, completely abandon
simulated annealing, devise 3 methods and explain each."*

**DONE.** New default engine rule (non-SA) beats the favorite on Week 2026-11-01:

- **Strategy: 11/12 (91.7%)**  vs  **Favorite: 10/12 (83.3%)** → **BEATS FAVORITE ✅**
- The one extra: correctly flipped **LV @ NYJ** (LV won 38-33). Correctly held KC@DEN (trap) and DAL@ARI (honest miss — data doesn't support the flip).
- **Simulated annealing: abandoned** as the decision layer. (Legacy path kept for A/B: `--strategy engine`.)

### The shipped rule — `strong_underdog.py` (pure, 13 tests, deterministic)
```
flip favorite -> underdog  IFF
    |spread| <= 3.0   (close game)
    AND (underdog_o_pp - favorite_o_pp) >= 0.20
otherwise keep favorite.
```
- 3-method comparison in `methods_eval.py`: M1 (offense+keyplayer veto)=10, M2 (logistic)=10,
  M3 (multi-signal vote)=10, **M4 (strong-underdog-offense)=11 (WINNER, stable across 3 thresholds)**.
- Full suite: **125 tests passing**.
- CLI: `python pickem.py --week 20261101 --offline --data-dir data` → prints Strategy vs Favorite.
- Report: `reports/04_beat_favorite.md`.

### ⚠️ HONEST CAVEAT (told Lord Bryson)
Validated on ONE week (12 games). Best-of-several + stable across 3 threshold settings + signal is
pre-game (oPP) not the score — but "11-10 on one week" is not proof of skill. A multi-week
backtest is the honest next step — **but 2024/2025 have no cached odds** (see Milestone 2),
so the "beat-the-favorite" backtest can only be run on the 2026 cached weeks unless Lord Bryson
supplies a historical-odds source.

## Key files
- `strong_underdog.py` — the rule (default)
- `consensus/{experts,judge,pipeline}.py` — 3-expert + agentic-judge LLM layer (NEW)
- `pickem.py` — CLI (`--strategy strong_underdog|engine|consensus`)
- `methods_eval.py` — M1–M4 evaluation harness
- `tests/test_strong_underdog.py` (13) + `tests/test_consensus.py` (23) — 148 total, green
- `poc/sim_engine/` — legacy SA+Monte Carlo (kept for A/B only)
- `reports/04_beat_favorite.md`, `reports/05_consensus.md` — writeups
- `data/` — cached ESPN data (2024,2025,2026) + `data/consensus_log/` (audit trail)

## R10 — Historical pick record + accuracy KPI (2026-11-08, Lord Bryson)

**Status: DONE, committed (`8ef9534`), pushed to `slbryson/nfl-pickem` (private).**

New durable local ledger + a dependency-free SVG/HTML dashboard, both committed so the
history ships with the repo.

- `data/history.json` — local accuracy ledger (committed). Week keying `2026W8`, `2026W9`…
  Idempotent upsert keyed by `(season, week, strategy)`; last-write-wins.
- `reports/accuracy_dashboard.html` — committed KPI artifact. Self-contained HTML + 2 SVGs,
  no JS, no CDN, no external deps. Opens in any browser.
- `charts.py` — dependency-free SVG/HTML rendering (line + bar + dashboard).
- `scoreboard.py` — ledger + KPI engine: load/save, record (upsert + scoring + Brier +
  confidence + flips), seed (Week 8), KPIs, chart-row extraction.
- `consensus/{experts,judge,pipeline}.py` — token plumbing end-to-end
  (`LLMClient.last_usage` → `ExpertResponse.tokens` / `JudgeResponse.tokens` →
  `ConsensusResult.tokens`).
- `pickem.py` — CLI flags `--record`, `--seed-ledger`, `--render`, `--label`, `--ledger`.
- `tests/test_scoreboard.py` (24) + `tests/test_charts.py` (11) — new unit tests.
- **Full suite: 211 passed.**
- `PROJECT_PLAN.md` — R10 requirement contract (schema, week keying, token plumbing, DoD).
- `README.md` — R10 section, quick commands, token plumbing notes.
- Skill: `/home/sidney/.openclaw/workspace/skills/nfl-pickem/SKILL.md` updated with the R10
  commands + KPI explanation.

**Week 8 seed (verified against the audited Nov-3 reports, not hand-derived):**
- consensus 9/12 (misses: ARI@DAL, KC@DEN, MIA@NE) → 75.0%
- strong_underdog 11/12 (miss: ARI@DAL) → 91.7%

**In-service command (unchanged):** `python pickem.py --slate [--present]`. R10 is
default-on after a scored week: `python pickem.py --slate --record`.

**Repos:**
- `github.com/slbryson/nfl-pickem` (private) — engine + ledger + dashboard + this status doc.
- `github.com/slbryson/brysonclaw` (public) — the `nfl-pickem` skill (workspace mirror).

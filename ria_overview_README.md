# RIA Risk Copilot

A risk-insight and client-explanation tool for small registered investment advisors (RIAs).

It loads an advisor's holdings across many accounts, computes portfolio risk deterministically, ranks accounts by severity, explains the findings to the advisor, and drafts a plain-English client note. The advisor edits and approves the note before anything reaches a client.

**It does not recommend or execute trades.** It flags risks and explains them. Any next step is framed as a topic for the advisor to review with the client.

> Built on the deterministic quant core of my earlier Portfolio Risk & Rebalancing Agent.

Source code is in a private repository. I'm happy to walk through it or share access on request.

## Why I built this

Small firms make up most of the advisory industry. Of the 16,544 SEC-registered advisers, about two-thirds manage under $1B, and they rely on spreadsheets, rule-based tools, or costly all-in-one suites. The tools built for them mostly rebalance and execute trades (Schwab's iRebal does it at no added cost), but none of them explain risk in a way an advisor can pass on to a client.

The first version of this project recommended trades. I changed course: rebalancing is already commoditized, and an advisor has to trust anything that goes in front of a client. So this version flags and explains risk, drafts the client note, and leaves every decision to the advisor.

## At a glance

- Six deterministic risk checks per account (concentration, sector, drift, correlated clusters, stress test, drawdown), ranked by severity
- Claude drafts an advisor summary and a client note from an allowlisted JSON payload, never from raw data or client names
- Every draft passes a number check and a compliance check, re-run on edit and on approval
- Full audit trail in SQLite: inputs hash, model, prompt version, every LLM attempt, edits, approver
- Streamlit app with demo data; runs fully offline with a template writer when no API key is set
- 65 offline tests, including hand-built portfolios with known answers and an app walkthrough

## Design principle: the LLM explains, it never calculates

```
holdings.csv ─┐
targets.csv  ─┼─► inputs.py ─► flags.py + risk_metrics.py ─► per-account JSON ─► llm.py ─► checks.py ─► advisor
config.yaml  ─┤   (validate)   (deterministic quant core)    (every number)      (words)   (verify)     edits/approves
prices       ─┘                                                                                         │
                                                                                         audit.py (SQLite) ◄┘ export.py (txt/pdf)
```

- **Every number is computed in Python.** That includes weights, drift, concentration, correlations, beta, drawdown, the stress test and severity.
- **The LLM receives only an allowlisted subset of the per-account JSON.** It never sees price data, household IDs or client names; accounts are identified by account ID only. Any extra holdings column, such as a client name, is dropped when the file is loaded.
- **Number check.** After generation, every number in the draft is extracted and must match a number in the JSON the model received. Matching allows for rounding to the displayed precision and for `%`, `$`, "million" and "percent" formatting. If any number fails, the draft is regenerated once, with the failing numbers named. If it fails again, it is returned with those numbers highlighted for the advisor.
- **Compliance check.** Configurable phrases and regexes ("guaranteed", "will outperform", "risk-free", "you should buy/sell", ...) are flagged.
- **Re-checked on edit and approval.** Both checks run again when the advisor edits and when they approve. Approving with open warnings needs an explicit acknowledgement, which is logged.
- **Recorded per note:** the model name and prompt version. Prompts are versioned files in `prompts/`.

## What it computes (per account)

| Check | Metric | Threshold (config.yaml) |
|---|---|---|
| Single-position concentration | position weight | `max_position_weight` |
| Sector concentration | sector weight | `max_sector_weight` |
| Drift from target | current − target, by position (ticker targets) or asset class | `drift_tolerance` |
| Correlated cluster | holdings where **every pair** correlates ≥ threshold (complete linkage, so A~B~C don't chain), even across sectors | `correlation_cluster_threshold`, `max_cluster_weight` |
| Stress test | estimated change = −beta × market drop | `stress_market_drop`, `max_stress_loss` |
| Max drawdown | worst peak-to-trough loss of today's weights over the lookback | `max_drawdown_limit` |

**Severity** depends on how far a value is past its threshold (`|value| / threshold`):

- up to `medium_multiple` (1.25×) → **low**
- up to `high_multiple` (1.5×) → **medium**
- above that → **high**

Each account's overall score is its worst flag. Accounts are ranked by that, then by severity points.

Diversified funds (`security_type: fund` in the security master) and cash are exempt from the single-name concentration and cluster checks, because a 35% VTI position is not single-stock risk. Set `policy.exempt_diversified_funds` to change this.

Output per account: a JSON object with every flag's `flag_id`, `type`, `metric`, `value`, `threshold`, `severity`, `holdings` and `details`, plus flag counts, the overall severity, metrics, drift tables and positions.

## Inputs

| File | Columns / shape |
|---|---|
| Holdings CSV | `account_id, household_id, ticker, quantity, market_value, cost_basis` (weights come from `market_value`) |
| Targets CSV | `account_id, target_type (asset_class \| ticker), key, target_weight`. You can also use JSON: `{"A-1": {"asset_class": {"US Equity": 0.6, "Bonds": 0.4}}}` |
| Config YAML | thresholds, severity bands, market data source, LLM model and prompt versions, compliance phrases (see `demo_data/config.yaml`) |
| Security master CSV | `ticker, sector, asset_class[, security_type]` (optional upload; defaults to `demo_data/securities.csv`) |

Bad files get specific errors, such as a missing column by name, a row number with a bad value, or targets that don't sum to 1.

## Market data

`market_data.source: snapshot` (the default) reads `demo_data/prices_snapshot.csv`, so the demo is offline and reproducible. `live` pulls from yfinance.

yfinance is an unofficial wrapper, fine for a demo. **A commercial version would use a licensed market-data provider**, and each yfinance call site carries a comment saying so.

The bundled snapshot may be **simulated** (factor model, fixed seed; see `scripts/make_simulated_snapshot.py`). The app shows a banner when it is. Run `scripts/build_price_snapshot.py` on a machine that can reach Yahoo Finance to replace it with real prices.

## Demo walkthrough (2 minutes)

1. **Upload.** On *Accounts*, upload holdings, targets and config, or click **Use demo data**.
2. **Ranked list.** Five accounts are sorted by overall severity, with flag counts, max drift, top position and top sector.
   - **A-1001**: tech-heavy. NVDA is 22% and Technology is 58%, and the account has drifted from its ticker-level model.
   - **A-1002**: large drift from a 60/40 target. Equities are overweight and bonds are 18 points under.
   - **A-1003**: the **hidden cluster**. No single position or sector breaches a limit. But NVDA, VRT, ETN, CEG and VST span Technology, Industrials and Utilities and all move together, making up 40% of the account.
   - **A-1005**: high beta. It fails the stress test and has a large drawdown.
   - **A-1004**: healthy, with no flags.
3. **Open the worst account.** Click its row.
4. **Flags.** Each flag shows its value against the threshold, severity and holdings. The computed JSON is one click away.
5. **Client note.** Click *Generate*. You get a technical advisor summary and a plain-English client note, each with its number-check and compliance-check result.
6. **Edit.** Personalize the note and save. The checks re-run on your text.
7. **Approve.** Enter your name in the sidebar and click *Approve*. You can then export the note as .txt or PDF.
8. **Audit log.** Every note's timestamp, input file hash, metrics JSON, model, prompt version, raw LLM output (every attempt), check results, edit diffs, approver and approval time.

## Approval workflow

`draft → edited → approved → exported`. Edits are allowed only before approval, and approved notes are locked. Everything is stored in SQLite (`ria_audit.db`, gitignored): a `notes` table and an append-only `events` table.

## Project layout

```
ria_app.py                 Streamlit entry (3 pages in ria_pages/)
ria_copilot/
  config.py                load/validate config.yaml (no hard-coded thresholds)
  inputs.py                holdings / targets / security-master validation, PII column drop
  market_data.py           snapshot or live (yfinance) prices
  flags.py                 drift, concentration, clusters, drawdown, stress, severity -> JSON
  llm.py                   payload allowlist, Claude call, regenerate-once, offline writer
  checks.py                number check, compliance check, highlighting
  audit.py                 SQLite workflow + audit trail
  export.py                txt / PDF export of approved notes
  pipeline.py              files -> ranked per-account JSON
risk_metrics.py            original quant core (+ max drawdown, portfolio returns, clusters)
prompts/                   advisor_summary_v1.md, client_note_v1.md
demo_data/                 holdings, targets, securities, config, price snapshot
scripts/                   build_price_snapshot.py, make_simulated_snapshot.py, verify_notes.py
tests/                     unit tests (hand-made portfolios with known answers) + AppTest walkthrough
legacy/                    original rebalancing agent (unchanged)
```

## Known limitations

- **The stress test is linear and single-factor** (beta × drop). It ignores convexity and the way correlations rise in a crisis.
- **Drawdown and correlation are backward-looking** over one lookback window. They describe the past, not a forecast.
- **The number check verifies that numbers come from the computed data, not that the sentence around them is right.** For example, a number could be attached to the wrong holding. That is why a human approves every note.
- **Numbers written as words ("two flags") aren't checked.**
- **No login, billing, trade execution or custodian integration, by design.**

## Author

Built by Riya Swayampati, MBA candidate at Cornell Tech (2027).

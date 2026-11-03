# DA-04 Financial Analytics and Risk Reporting System

An automated reporting pipeline that produces a monthly profit and loss statement, a budget-versus-actual variance analysis with exception flags, a credit risk score for every customer, and a board-pack summary. It ships as one Python pipeline, a PostgreSQL schema, and a self-contained HTML dashboard that opens offline in any browser.

Nexvia Technologies capstone, Data Analytics track. Author: Mayank Swaraj.

> **Data note:** all data is synthetic (FY2025 ledger, 1,000 customers, amounts in INR). Two issues are injected on purpose so the variance engine can be verified: a **Marketing overspend** (Jul to Sep) and a **Services revenue shortfall** (Jul to Dec).

## Contents
1. [Results at a glance](#results-at-a-glance)
2. [Quick start](#quick-start)
3. [Dashboard tour](#dashboard-tour)
4. [Architecture](#architecture)
5. [How it works](#how-it-works)
6. [Configuration](#configuration)
7. [Project structure](#project-structure)
8. [Data dictionary](#data-dictionary)
9. [Loading into PostgreSQL](#loading-into-postgresql)
10. [Testing](#testing)
11. [Verification status](#verification-status)
12. [Limitations](#limitations)
13. [Roadmap](#roadmap)
14. [Requirement map](#requirement-map)

## Results at a glance
Output of `python pipeline.py` with the default seeds.

**FY2025 P&L (INR)**

| Line | Actual | Budget | Variance |
|---|---|---|---|
| Revenue | 10,485,397 | 10,634,400 | -1.4% |
| COGS | 4,877,106 | 4,810,800 | +1.4% |
| Gross profit | 5,608,291 | 5,823,600 | -3.7% |
| Operating expenses | 4,623,469 | 4,595,580 | +0.6% |
| Net income | 984,823 | 1,228,020 | -19.8% |

**Variance flags (favourable variance below -5%)**

| Account | Variance | Flag |
|---|---|---|
| Services | -6.0% | Below target |
| Marketing | -6.0% | Over budget |

No other account crossed the threshold, so both injected issues are found and nothing else is flagged.

**Credit scorecard**

| Metric | Value |
|---|---|
| Hold-out AUC | 0.771 (300 held-out customers, 700 for training) |
| Observed default rate | 12.0% |
| Risk bands | Low 255, Medium 304, High 255, Very High 186 |
| High + Very High | 441 customers (44.1%) |

## Quick start
```bash
pip install -r requirements.txt
python pipeline.py            # writes data/*.csv, docs/metrics.json, dashboard.html
python -m pytest tests        # 6 tests
open dashboard.html           # or double-click; works offline in any browser
```

Requirements: Python 3.10+. `matplotlib` and `reportlab` are only needed to rebuild the research paper (`python build_paper.py`).

## Dashboard tour
`dashboard.html` is one file with the data embedded, so it needs no server or internet. It follows your system light/dark mode.

| Tab | What it shows |
|---|---|
| Overview | KPI cards (revenue, gross profit, net income, net vs budget, budget exceptions) and a monthly revenue and net income chart |
| P&L | Monthly actual P&L table (revenue, COGS, gross profit, opex, net income) with a full-year total row |
| Budget vs Actual | Budget and actual chart by account; table with favourable variance, variance % and flag |
| Credit Risk | Customer count, default rate, AUC, band chart, scorecard drivers, and the 10 highest-risk customers |
| Board Pack | Auto-written summary, key financials and budget exceptions; the Print / Save as PDF button prints only this page |

## Architecture
```
 gen_ledger() ──────────┐                         ┌─> P&L (monthly, actual vs budget)
                        ├─> data/gl_ledger.csv ──>├─> Budget vs actual + flags (>5% adverse)
 gen_customers() ─┐                               └─> Board-pack narrative
                  └─> fit_scorecard() ─> data/customers_scored.csv ─> bands, top 10, drivers
                                  everything ─> dashboard_template.html ─> dashboard.html
```
More detail in `docs/architecture.md`.

## How it works

### 1. Ledger simulation (`gen_ledger()`)
- Nine accounts over the twelve months of 2025: Product Sales, Services and Subscriptions (revenue), Cost of Goods Sold, and five operating expenses (Salaries, Marketing, Rent, Software, Travel).
- Budget for month `i` is the account's base value grown by 1% per month.
- Actual is budget times a seasonal factor times noise: `actual = budget × season × (1 + e)`, with `season = 1 + 0.12 × sin(2π(i-2)/12)` (applied to revenue and COGS only) and `e ~ N(0, 0.04)`.
- Injected issues: Marketing × 1.25 in Jul to Sep; Services × 0.88 from Jul onward.

### 2. P&L and variance rules (`pnl()`, `budget_vs_actual()`)
- Gross profit = revenue - COGS. Net income = gross profit - operating expenses. Computed for actual and budget in every month.
- Favourable variance is `actual - budget` for revenue and `budget - actual` for costs, shown as a percentage of budget.
- An account is flagged **Over budget** (cost) or **Below target** (revenue) when its favourable variance is below -5%.

### 3. Credit scorecard (`gen_customers()`, `fit_scorecard()`, `to_score()`)
- 1,000 customers with five features: income (log-normal), debt ratio (Beta(2,5)), late payments (Poisson(1.2)), utilisation (Beta(2,3)), tenure in years (1 to 14).
- Defaults are drawn from `sigmoid(z)` with `z = -3.2 + 3.0·debt_ratio + 0.45·late_payments + 1.8·utilization - 0.08·tenure - 0.00001·income`.
- 70/30 split by a seeded permutation; features standardised on the training set; L2-regularised logistic regression (scikit-learn defaults). AUC is measured on the 30% hold-out.
- Points scale: `score = 650 + (40 / ln 2) × ln((1 - PD) / PD / 20)`, clipped to 300 to 850. That is 650 points at 20:1 good-to-bad odds and 40 points per doubling of odds.
- Bands: **Low** 640 and above, **Medium** 600 to 639, **High** 560 to 599, **Very High** below 560. The cut-offs were set after inspecting the score distribution so all four bands are populated.

| Feature | Standardised coefficient |
|---|---|
| late_payments | +0.518 |
| debt_ratio | +0.493 |
| tenure_years | -0.365 |
| income | -0.315 |
| utilization | +0.026 |

### 4. Board-pack narrative (`narrative()`)
Generated from the computed figures: full-year revenue and net income against budget, the three worst flagged accounts, and a credit-book summary. Every sentence comes from computed values, so the text cannot drift from the numbers.

## Configuration
There are no environment variables. Settings are constants in `pipeline.py`:

| Setting | Where | Default |
|---|---|---|
| Accounts and base amounts | `ACCTS` | 9 accounts |
| Months | `MONTHS` | 2025-01 to 2025-12 |
| Scorecard features | `FEATURES` | income, debt_ratio, late_payments, utilization, tenure_years |
| Risk band cut-offs | `BANDS` | 640 / 600 / 560 |
| Variance flag threshold | `budget_vs_actual()` | -5% favourable variance |
| Seeds | `gen_ledger(seed=42)`, `gen_customers(seed=7)`, `fit_scorecard(seed=1)` | 42, 7, 1 |

## Project structure
```
DA-04/
├── pipeline.py                  data generation, scorecard, P&L, variance, narrative, dashboard build
├── dashboard_template.html      UI template (data injected at build)
├── dashboard.html               generated, self-contained dashboard
├── schema.sql                   PostgreSQL tables for the two CSVs
├── tests/test_pipeline.py       6 pytest tests
├── data/                        gl_ledger.csv, customers_scored.csv
├── docs/                        architecture, metrics, test results, presentation outline, reflection
├── build_paper.py, paperkit.py  rebuild the research paper PDF
├── DA-04_Research_Paper_and_Source.pdf
└── requirements.txt
```

## Data dictionary
| Table | Columns |
|---|---|
| `gl_ledger` | month, account, type (Revenue / COGS / Opex), budget, actual |
| `customers_scored` | id, income, debt_ratio, late_payments, utilization, tenure_years, defaulted, pd, score, band |

## Loading into PostgreSQL
```bash
psql -d yourdb -f schema.sql
psql -d yourdb -c "\copy gl_ledger FROM 'data/gl_ledger.csv' CSV HEADER"
psql -d yourdb -c "\copy customers_scored FROM 'data/customers_scored.csv' CSV HEADER"
```
This step is optional; the dashboard does not need a database.

## Testing
```bash
python -m pytest tests -v
```
| Test | Checks |
|---|---|
| ledger shape and determinism | 12 months × 9 accounts, identical output for the same seed |
| P&L identity | net = revenue - COGS - opex and gross = revenue - COGS, every month, to 1e-6 |
| injected variance issues | Marketing flagged Over budget, Services flagged Below target |
| scorecard quality and ranges | AUC above 0.70, scores within 300 to 850, valid band labels |
| score monotonicity | a lower PD always gives a higher score |
| default rate | observed default rate between 5% and 25% |

Result: 6 passed. The dashboard JavaScript was also executed against a stubbed browser environment to catch runtime errors.

## Verification status
| Component | Status |
|---|---|
| Pipeline, scorecard, P&L, variance flags, narrative | Run and tested |
| Dashboard | JavaScript smoke-tested; open it in a browser to confirm the visuals and the print layout |
| Research paper PDF | Built and visually checked |
| `schema.sql` and the PostgreSQL load commands | **Written but not run against a database** |

## Limitations
- **Synthetic data.** Absolute figures carry no business meaning; the project shows the pipeline is correct and reproducible.
- **Circular AUC.** Defaults were generated from a logistic process, so logistic regression is the right model by construction. Read the 0.771 AUC as an implementation check, not evidence of real-world predictive power.
- **Estimation noise.** Only about 80 defaults inform the fit. Utilisation was generated with a positive effect, yet its fitted coefficient is near zero.
- **In-sample scoring.** Dashboard scores cover all customers including the training set; only the AUC uses held-out data.
- **Board pack scope.** The board pack summarises the full year, and the monthly view is in the P&L tab. It is a printable HTML page, not an automatically generated PDF file.
- **Illustrative thresholds.** The 5% variance limit and the band cut-offs should be set from materiality and policy in a real deployment.
- **Not included.** Expected-credit-loss provisioning, fairness testing of the scorecard, multi-entity consolidation.

## Roadmap
Load real ledger exports through the PostgreSQL schema; run the pipeline on a monthly schedule; add rolling forecasts alongside budget; calibrate PD and validate the scorecard out of time; add expected-credit-loss estimates; generate the board pack directly as a paginated PDF.

## Requirement map
| Project requirement | Where |
|---|---|
| Automated P&L | `pnl()`, P&L tab |
| Budget vs actual | `budget_vs_actual()`, Budget vs Actual tab |
| Credit risk scores | `fit_scorecard()`, `to_score()`, Credit Risk tab |
| Monthly board pack | `narrative()`, Board Pack tab (print to PDF) |
| Research and documentation | `DA-04_Research_Paper_and_Source.pdf`, `docs/` |

## License
Released under the MIT License. Copyright (c) 2026 Mayank Swaraj. See [LICENSE](LICENSE).

MIT License

Copyright (c) 2026 Mayank Swaraj

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

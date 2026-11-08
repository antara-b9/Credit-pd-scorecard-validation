# PD Scorecard Development and Independent Validation (Lending Club)
Independent portfolio project using public data only. Not employer work.
Built with AI assistance for code drafting; all methodology choices, results and findings were reviewed by me.
Results are for learning and should not be read as production performance.

## Purpose
Build a probability-of-default scorecard for 36-month Lending Club loans and validate it as an
independent reviewer would. The aim is to show how a credit scorecard is built and then challenged: out-of-time testing, a challenger model, stability checks and risk-rated findings.

## Data
- Lending Club accepted loans, 2007-2018 Q4 (2,260,668 loans after removing footer lines).
- Default definition: Charged Off or Default = bad; Fully Paid = good; Current, Late, In Grace
  Period and policy-exception loans excluded (final outcome not known).
- 36-month loans only (60-month loans behave differently, and later 60-month vintages are unfinished).
- Development sample: 2013-2014 issue years (262,992 loans, 13.2% default).
- Out-of-time sample: 2015 (283,026 loans, 14.9% default).
- Excluded: 2012 (several bureau fields were collected only partway through the year: 51.9% missing),
  2016-2018 (loans had not reached final outcomes by the end of the data).

## Method
1. Removed 42 columns known only after loan issue (leakage) and 35 columns missing more than 50%.
2. Capped outliers at the development 99th percentile (income, revolving balance) and 100 (utilisation).
3. Screened variables by information value; removed near-duplicates (correlation above 0.8).
4. Binned variables by weight of evidence (monotonic, up to 8 bins, missing as its own bin).
5. Fitted a logistic regression on WoE values: 14 variables after removing credit_hist_years (wrong
   sign caused by 0.85 correlation with mo_sin_old_rev_tl_op).
6. Scaled to points: 600 points = 50:1 good-to-bad odds, 20 points to double the odds.
7. Challenger: XGBoost (up to 600 trees, depth 4, early stopping on 20% of development data).

## Results (out-of-time, 2015)
| Model | Gini | KS |
|---|---|---|
| Scorecard | 0.331 | 0.239 |
| XGBoost challenger | 0.351 | 0.253 |
| Benchmark: Lending Club interest rate | 0.357 | 0.262 |
- Score PSI development vs 2015: 0.0035; all variables below 0.026 (stable).
- Calibration: predicted default 13.5% vs actual 14.9%; under-prediction in 9 of 10 deciles.

## Findings
## Findings

| ID | Area | Finding | Severity | Recommendation | Status |
|---|---|---|---|---|---|
| F1 | Calibration | Both models under-predict 2015 defaults: 13.5% (scorecard) and 13.7% (XGBoost) vs 14.9% actual; actual defaults exceed predicted in 9 of 10 deciles. Population mix was stable (score PSI 0.0035), so the gap reflects higher default behaviour in 2015. | Medium | Recalibrate the intercept on more recent data; monitor calibration each quarter. | Open |
| F2 | Challenger benchmarking | XGBoost improves out-of-time Gini by about 0.020 (0.351 vs 0.331); added complexity is not justified. | Low | Keep the scorecard as the primary model; revisit if the gap widens. | Open |
| F3 | Stability over time | Challenger Gini falls from 0.376 (development) to 0.351 (2015), indicating some period sensitivity. | Low | Monitor challenger performance if it is ever promoted. | Open |
| F4 | External benchmark | Scorecard trails the lender's interest rate by about 0.026 Gini (0.331 vs 0.357). | Low | Informational: the rate uses information outside the 15 variables. | Open |
| F5 | Variable selection | Wrong-sign coefficient on credit_hist_years (+0.406) caused by 0.85 correlation with mo_sin_old_rev_tl_op; variable removed and model refitted with no loss of performance (Gini 0.316 / 0.331 unchanged). | Low | Closed during development; keep correlation screening in the build procedure. | Closed |

Full log: `findings_log.csv`.

## Limitations
- Public data; FICO scores are all 660 or above; no information on the lender's own underwriting.
- The model ranks well but understates the level of risk in 2015 (default rates rose).
- Variables such as income and home ownership may act as proxies; no fairness testing done here.
- Not a production model; no real-time deployment.

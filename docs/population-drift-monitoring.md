# Population drift monitoring

**Status: a documented review policy and an offline stress test. Production drift monitoring is not implemented.** Do not describe the dashboard as monitored for drift until the first section's "Not implemented" rows are closed.

## What exists

| Control | Where | Status |
| --- | --- | --- |
| PSI on one feature, 10 quantile bins | `population_stability_index` in `ml/train_census_forecast.py` | Implemented. Called only by the offline stress test. |
| Level-shift stress test, 0% to 30% on lagged census | `drift_stress_test` in the same file; committed as `ml/artifacts/drift_stress_test.csv` | Implemented. Synthetic data only. |
| Review threshold, PSI at or above 0.20 | `drift_retrain_psi_threshold` in `ml/artifacts/backtest_metrics.json`; census model card | Documented policy. No code reads it at runtime. |
| Retraining criteria | `drift_policy` in `backtest_metrics.json`; census model card | Documented policy. No automated trigger. |
| Sheet 16 drift chart | `app.py`, "Synthetic Drift Stress Test" | Displays the committed stress-test artifact. Not live data. |
| Scheduled PSI on incoming census | none | **Not implemented.** |
| Alerts, named owner, drift log | none | **Not implemented.** |
| Drift monitoring for the CMS classifier | none | **Not implemented.** Its maintenance rule is a source-version review (`models/cms/model_card.md`). |

**Gap between policy and code.** The census model card says "Monitor PSI on lagged census features". The only PSI computation in the repository is on `lag_7`. The lags at 1, 14 and 28 days, and the 7- and 28-day rolling means, are not monitored.

## Scope

- **Census forecast** (`ml/artifacts/model_card.md`): a Ridge model that forecasts 30 days of synthetic daily census. Its inputs include lags at 1, 7, 14 and 28 days and trailing 7- and 28-day means. Drift here means a change in the distribution of these inputs.
- **CMS readmission classifier** (`models/cms/model_card.md`): a cross-sectional model over published CMS attributes. It is not a forecast. "Population" means the hospitals in each CMS extract. Its model card allows a refresh only after source-version review, grouped revalidation, subgroup comparison and manifest approval.

## PSI method (as implemented)

- **Reference:** the training-period distribution of the feature. In the stress test, that is `lag_7` from the training rows.
- **Bins:** up to 10 quantile edges of the reference. The outer edges are minus and plus infinity, so out-of-range values fall into the end bins. Duplicate quantile edges are merged, so a discrete feature can produce fewer than 10 bins.
- **Score:** PSI = sum over bins of (actual − expected) × ln(actual ÷ expected). Bin proportions are floored at 1e-6, so empty bins do not produce infinities.
- **Threshold:** 0.20. The repository defines only this review threshold. It defines no lower bands.

## Observed behavior (committed stress test)

The stress test multiplies every lagged census feature and the target by `1 + shift`, then scores PSI on `lag_7` and the Ridge model's MAE. The seasonal-naive baseline MAE is 8.48 beds.

| Synthetic level shift | PSI on `lag_7` | Ridge MAE (beds) | Worse than seasonal-naive? |
| --- | --- | --- | --- |
| 0% | 0.00 | 6.09 | No |
| 5% | 0.64 | 6.49 | No |
| 10% | 2.39 | 7.02 | No |
| 15% | 5.14 | 7.65 | No |
| 20% | 8.23 | 8.36 | No |
| 25% | 8.35 | 9.15 | Yes |
| 30% | 7.20 | 9.99 | Yes |

What this shows:

- **The review trigger fires early.** A 5% synthetic shift crosses the 0.20 threshold (PSI 0.64) while error is still well below the seasonal baseline. Error first becomes worse than seasonal-naive at a 25% shift. That matches the documented policy: review on a shift, retrain only on sustained degradation.
- **PSI is not monotonic at large shifts.** It falls from 8.35 at 25% to 7.20 at 30%, because the fixed reference bins saturate. A single-feature PSI can understate extreme shifts, so it should not be the only alarm for large changes.
- **These are deterministic synthetic shifts.** They show how the tool behaves. They do not estimate how often real drift occurs.

## Response procedure (documented)

1. PSI at or above 0.20 on a monitored lagged census feature triggers review and prospective error validation (census model card, "Drift and maintenance").
2. Retrain only after sustained prospective error degradation, or error worse than seasonal-naive (`drift_policy`).
3. A retrained model must pass the existing rolling-origin validation and the pre-registered rule: gradient boosting is adopted only if it cuts rolling-origin MAE by at least 5% against Ridge.

**Not defined in the repository:** the review owner, the escalation path, the monitoring cadence, and how many weeks count as "sustained". These need a decision before production monitoring is claimed.

## Proposed production monitoring (not implemented)

These are proposals for review. They do not describe current behavior.

- **Census:** compute PSI for every lagged census input (lags at 1, 7, 14 and 28 days, and the rolling means), not only `lag_7`. Compare each against the training reference on each new day or week of census data.
- **Census error:** track prospective MAE against seasonal-naive over a rolling window, so "sustained" degradation can be measured rather than judged.
- **Records:** store each PSI and MAE value with its date, the model artifact hash, and the threshold in force. Alert a named owner when the 0.20 threshold is crossed.
- **CMS classifier:** at each source refresh, compare feature distributions with the previous source version. Use the SHA-256 hashes that `etl/fetch_cms.py` already records for the source files and the feature table.

## Limitations

- No production drift monitoring exists. The census data is synthetic. Any monitoring claim must name the monitored features and the period it covers.
- The CMS classifier is cross-sectional and does not forecast. For it, "drift over time" means a change in the source snapshots, not in a stream of patients.
- Neither model is patient-care decision support. The census model card requires external validation before operational use, and the CMS model card rules out patient-care use.

## Files

- `ml/train_census_forecast.py`: PSI, the stress test, and the policy text written into the artifacts
- `ml/artifacts/drift_stress_test.csv`, `ml/artifacts/backtest_metrics.json`, `ml/artifacts/model_card.md`
- `app.py`: the Sheet 16 drift chart and its caption
- `models/cms/model_card.md`: the CMS maintenance rule

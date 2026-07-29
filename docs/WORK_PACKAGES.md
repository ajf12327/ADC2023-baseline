# Track A Work Packages

## Document status

This document is the canonical scope and execution plan for **Track A**, the light conformal-prediction phase of the Ariel ADC 2023 project.

It records two kinds of statements:

- **Agreed decision** — part of the current project scope.
- **Provisional** — a working choice that may be revised after data inspection or early experiments.

Track B and the wider research programme are intentionally outside the committed scope of this document.

---

## 1. Track A definition and purpose

### Agreed decision

Track A is a self-contained conformal regression study using the Ariel ML Data Challenge 2023 as its scientific test bed.

Its main research question is:

> Can split conformal prediction produce reliable and reasonably efficient uncertainty intervals for atmospheric parameters inferred from Ariel exoplanet spectra?

A secondary question is:

> How does conformal coverage change when the test spectra differ from the calibration distribution?

Track A will estimate the seven simulator parameters associated with each synthetic spectrum:

- planet radius;
- atmospheric temperature;
- log-abundance of H2O;
- log-abundance of CO2;
- log-abundance of CO;
- log-abundance of CH4;
- log-abundance of NH3.

The simulator or forward-model parameter vector is the Track A target. The resulting conformal outputs are to be interpreted as frequentist prediction intervals or regions for simulator parameters across repeated synthetic planets. They are not intended to reproduce the full Bayesian atmospheric posterior.

### Purpose

Track A is intended to:

1. establish a reproducible Ariel data and evaluation pipeline;
2. measure whether raw model uncertainty is calibrated;
3. compare standard split conformal regression with conformalized quantile regression;
4. distinguish marginal parameter coverage from simultaneous seven-parameter coverage;
5. quantify the efficiency cost of obtaining coverage;
6. demonstrate how ordinary conformal calibration behaves under one controlled distribution shift;
7. provide evidence for or against expanding into Track B.

---

## 2. Committed Track A scope

### Agreed decisions

Track A includes:

- one reproducible data pipeline;
- fixed train, validation, calibration and test partitions;
- simulator parameters as labels;
- one lightweight quantile-regression neural model;
- raw quantile intervals;
- marginal split conformal regression;
- conformalized quantile regression;
- one simultaneous seven-parameter conformal method;
- naive marginal and Bonferroni joint-coverage baselines;
- one controlled distribution-shift experiment;
- marginal coverage, simultaneous coverage and interval-efficiency evaluation;
- one subgroup or difficulty diagnostic;
- reproducibility, leakage and conformal-quantile tests.

The primary nominal coverage level is **90%**.

---

## 3. Deliberate exclusions

### Agreed decisions

The following are not required for Track A:

- diffusion models;
- normalising flows;
- flow matching;
- full neural posterior estimation;
- training on MultiNest posterior traces;
- posterior-density conformity scores;
- sample-cloud or posterior-rank conformity scores;
- atmospheric forward-model calls during conformal calibration;
- physics-informed conformity scores;
- weighted conformal prediction;
- adaptive online conformal inference;
- conformal risk control;
- full or transductive conformal prediction;
- cross-conformal or CV+ methods;
- classification conformal methods such as APS or RAPS;
- time-series conformal prediction;
- multiple distribution-shift families;
- recreation or optimisation of the original challenge leaderboard solution.

These methods may be reconsidered only after Track A has produced a complete and interpretable result.

---

## 4. Work-package overview

| Work package | Scope | Main output |
|---|---|---|
| WP1 | Research specification | Frozen Track A protocol |
| WP2 | Data audit, splits and preprocessing | Validated, reproducible modelling dataset |
| WP3 | Quantile-regression baseline | Frozen predictive model and raw-interval results |
| WP4 | Marginal conformal calibration | Split-CP and CQR results |
| WP5 | Simultaneous calibration | Joint-coverage and efficiency comparison |
| WP6 | Conditional diagnostics | Groupwise coverage analysis and a Mondrian decision |
| WP7 | Controlled distribution shift | Coverage-degradation analysis |
| WP8 | Final analysis and Track B decision | Reproducible final evidence and recommendation |

---

## 5. WP1 — Research specification

### Scope

Define the statistical target, experimental protocol and boundaries of Track A before model development.

### Objectives

- Fix simulator parameters as the prediction target.
- Fix 90% as the primary coverage level.
- Define marginal coverage for each parameter.
- Define simultaneous coverage for the complete seven-parameter vector.
- Define interval efficiency as a required companion to coverage.
- Define the distinction between IID evaluation and shifted evaluation.
- Record the methods and tasks excluded from Track A.

### Deliverables

- Canonical scope document.
- Written definitions of all primary evaluation quantities.
- Initial experiment registry.
- Explicit list of excluded extensions.

### Acceptance criteria

WP1 is complete when:

- the Track A question can be stated without relying on Track B;
- simulator truth, coverage targets and data roles are unambiguous;
- every planned experiment maps to the Track A research questions;
- excluded methods are not dependencies of the core pipeline.

---

## 6. WP2 — Data audit, splits and preprocessing

### Purpose

WP2 creates the trusted data foundation for all later work. No modelling or conformal result is considered valid unless it uses the frozen outputs and rules established here.

### Agreed objectives

#### 6.1 Source-data audit

- Inspect the available Ariel spectra, spectral uncertainties, stellar and planetary metadata, and simulator parameters.
- Determine the exact number of usable examples from the downloaded data rather than relying on hard-coded counts from external descriptions.
- Verify the dimensionality and ordering of the spectral channels.
- Verify that target and feature identifiers align across source files.
- Record the exact feature and target fields used by Track A.

#### 6.2 Integrity validation

The preparation pipeline must check that:

- each example contains the expected number of wavelength bins;
- identifiers align across all selected sources;
- each selected target is present;
- selected features and targets contain no unhandled NaN or infinite values;
- spectral uncertainties are positive;
- feature dimensions are consistent;
- target transformations are reversible;
- no example appears in more than one split.

The pipeline should fail clearly when an integrity condition is violated rather than silently discard malformed examples.

#### 6.3 Feature construction

The initial modelling input will include:

- spectral transit depths;
- spectral measurement uncertainties;
- selected stellar and planetary metadata available at prediction time.

When the wavelength grid is common to all examples, wavelength position may be represented by feature order rather than repeated as an input. If the audit finds varying wavelength grids, this assumption must be revisited.

#### 6.4 Target representation

- Retain molecular abundances in their supplied logarithmic representation.
- Standardise all seven targets using training-set statistics.
- Preserve inverse transformations so predictions and intervals can be reported in original units.

#### 6.5 Feature preprocessing

- Fit all preprocessing transformations on the training partition only.
- Standardise continuous spectral, uncertainty and metadata features as required by the baseline model.
- Apply the frozen transformations unchanged to validation, calibration and test data.

#### 6.6 Split protocol

Create and persist four mutually exclusive partitions:

1. **Training set** — model fitting.
2. **Validation set** — early stopping and limited model selection.
3. **Calibration set** — conformal-score and threshold estimation after the model is frozen.
4. **Test set** — untouched final evaluation.

The split must be made by unique planet identifier, use a fixed seed, and be saved as explicit identifier lists.

### Provisional choices

The following choices remain provisional until the data audit is complete:

- an initial 60% / 10% / 15% / 15% train-validation-calibration-test allocation;
- whether the common wavelength grid permits omitting wavelength values as explicit inputs;
- the exact metadata subset;
- whether simple global standardisation is sufficient or spectrum-specific normalisation is needed.

These choices must be resolved without using final test outcomes.

### Deliverables

WP2 must produce:

- a data-audit report containing exact counts, shapes and selected fields;
- a data dictionary for the Track A inputs and targets;
- validated processed arrays or equivalent serialised datasets;
- saved train, validation, calibration and test identifier lists;
- fitted preprocessing and target-transformation objects;
- a reproducible data-preparation command or script;
- automated tests covering split disjointness, dimensions, transformations and invalid values.

### Acceptance criteria

WP2 is complete when:

- one command recreates the processed Track A dataset from the available raw files;
- all selected examples pass the documented integrity checks;
- the four partitions are mutually exclusive and reproducible;
- preprocessing uses training data only;
- target transformations round-trip correctly;
- every later work package can consume the saved outputs without reading or interpreting raw source files independently;
- exact sample counts and dimensions are documented rather than assumed.

---

## 7. WP3 — Quantile-regression baseline

### Scope

Train one lightweight neural model that predicts lower, median and upper conditional quantiles for each of the seven targets.

### Agreed objectives

- Produce 0.05, 0.50 and 0.95 quantile predictions for each target.
- Use pinball loss for quantile training.
- Prevent or explicitly handle quantile crossing.
- Use validation data for early stopping and limited model selection.
- Freeze the selected model before accessing the calibration labels.
- Evaluate median point predictions and raw 90% quantile intervals.

### Provisional ideas

- Begin with a compact multilayer perceptron over the prepared features.
- Consider a small one-dimensional spectral encoder only if the simpler model is inadequate.
- Keep hyperparameter search deliberately small.

### Deliverables

- Frozen model configuration and checkpoint.
- Training and validation logs.
- Point-prediction metrics.
- Raw interval coverage and width results.
- Constant or naive predictor comparison sufficient to establish that the model is informative.

### Acceptance criteria

WP3 is complete when:

- predictions are finite for every evaluation example;
- lower, median and upper predictions are ordered;
- the model performs meaningfully better than a constant-median baseline;
- the checkpoint and preprocessing pipeline reproduce the reported predictions;
- no calibration or test outcome was used for model selection.

---

## 8. WP4 — Marginal conformal calibration

### Scope

Implement and compare two marginal methods for each atmospheric parameter:

1. split conformal regression using absolute residuals around the predicted median;
2. conformalized quantile regression using violations of the raw quantile interval.

### Objectives

- Implement the finite-sample conformal order statistic correctly.
- Produce calibrated 90% intervals for every target.
- Compare raw quantiles, split conformal and CQR on coverage and width.
- Report uncertainty in empirical coverage estimates.

### Deliverables

- Reusable split-CP and CQR implementations.
- Parameter-wise coverage and width tables.
- Coverage-versus-efficiency plots or equivalent figures.
- Unit test against a manually calculated conformal quantile.
- Toy IID regression test demonstrating approximately nominal coverage.

### Acceptance criteria

WP4 is complete when:

- the calibration set is used only to estimate conformity thresholds;
- the finite-sample quantile calculation is tested;
- IID test coverage is reported for all seven parameters;
- coverage is interpreted together with interval width;
- any material deviation from nominal coverage is investigated and documented.

---

## 9. WP5 — Simultaneous seven-parameter calibration

### Scope

Study the difference between marginal coverage and coverage of the entire atmospheric vector.

### Agreed methods

Compare:

- naive use of 90% marginal CQR intervals;
- Bonferroni-adjusted marginal intervals;
- one direct joint conformal score based on the maximum standardised interval violation across the seven targets.

The joint method will produce a rectangular seven-dimensional region. Modelling non-rectangular posterior geometry is outside Track A.

### Objectives

- Measure simultaneous coverage of ordinary marginal intervals.
- Calibrate a single score that targets joint coverage.
- Compare joint coverage with region efficiency.
- Show the cost of moving from marginal to simultaneous guarantees.

### Deliverables

- Simultaneous-coverage comparison.
- Joint-region efficiency metric and results.
- Method description and implementation of the max-score approach.
- Clear discussion of the rectangular-region limitation.

### Acceptance criteria

WP5 is complete when:

- simultaneous coverage is evaluated as an all-seven-parameters event;
- naive, Bonferroni and direct joint conformal approaches are compared on the same split and model;
- target-scale differences are handled in the joint score;
- joint coverage is reported alongside a region-size or width-based efficiency measure.

---

## 10. WP6 — Conditional diagnostics

### Scope

Determine whether acceptable global coverage hides systematic undercoverage for a physically or statistically meaningful subgroup.

### Agreed objectives

- Select one difficulty or grouping variable.
- Divide the evaluation data into a small number of groups.
- Report marginal and, where meaningful, simultaneous coverage by group.
- Decide whether a simple Mondrian extension is justified.

### Provisional grouping candidates

- spectral signal-to-noise;
- average observational uncertainty;
- predicted interval width;
- planet surface gravity.

Only one grouping strategy should be adopted for the core analysis.

### Deliverables

- Group-definition note.
- Groupwise coverage and width results.
- Written decision on whether to add a Mondrian calibration experiment.

### Acceptance criteria

WP6 is complete when:

- the grouping rule is defined without using test labels;
- each reported group contains enough examples for interpretable coverage estimates;
- subgroup findings are distinguished from formal conditional-coverage guarantees;
- Mondrian CP is added only if the diagnostics reveal a clear and adequately supported problem.

---

## 11. WP7 — Controlled distribution shift

### Scope

Evaluate the frozen Track A methods when the test distribution differs from the original calibration distribution.

### Agreed shift

The initial shift will be increased observational noise applied to the test spectra while leaving the atmospheric targets unchanged. The corresponding uncertainty inputs must be updated consistently.

### Objectives

- Construct a reproducible shifted test set.
- Keep the trained model and original conformal thresholds frozen.
- Measure changes in point error, marginal coverage, simultaneous coverage and interval width.
- Identify which parameters are most affected.
- Interpret the result as a stress test, not as a preserved conformal guarantee.

### Provisional choice

The main noise-inflation factor has not yet been fixed and must be declared before the final shifted evaluation.

### Deliverables

- Shift-generation implementation and configuration.
- Validation that targets are unchanged and uncertainties are updated.
- IID-versus-shift result tables and figures.
- Explicit discussion of exchangeability failure and the limits of ordinary split CP.

### Acceptance criteria

WP7 is complete when:

- the shifted set is reproducible;
- no shifted labels are used for recalibration;
- the original model and thresholds are reused unchanged;
- changes in coverage and efficiency are quantified;
- conclusions are limited to the defined noise shift and do not claim general robustness.

---

## 12. WP8 — Final analysis and Track B decision

### Scope

Consolidate the evidence, assess robustness to experimental randomness, and decide whether a richer posterior model is justified.

### Objectives

- Repeat final experiments over a small number of independent seeds when computationally practical.
- Report coverage uncertainty.
- Produce final tables and figures from reproducible outputs.
- Map conclusions to the predefined hypotheses.
- Record limitations and unresolved questions.
- Make a documented Track B recommendation.

### Provisional choice

Three to five split or training seeds were discussed as a possible final robustness check; the final number remains dependent on computational cost.

### Deliverables

- Final results package.
- Reproducibility instructions.
- Conclusions linked to the Track A questions.
- Track B decision note.

### Acceptance criteria

WP8 is complete when:

- every main claim is supported by a predefined metric and reproducible experiment;
- coverage is always reported with efficiency;
- marginal and simultaneous conclusions are kept distinct;
- IID and shifted conclusions are kept distinct;
- limitations are explicit;
- the Track B recommendation is based on observed Track A limitations rather than model novelty alone.

---

## 13. Intended sequence of work

### Agreed sequence

1. **WP1 — Freeze the protocol.**
2. **WP2 — Build and validate the data foundation.**
3. **WP3 — Train and freeze the quantile baseline.**
4. **WP4 — Calibrate and evaluate marginal intervals.**
5. **WP5 — Evaluate simultaneous coverage.**
6. **WP6 — Diagnose subgroup reliability.**
7. **WP7 — Run the controlled shift experiment.**
8. **WP8 — Consolidate results and decide on Track B.**

WP4 must not begin until the WP3 model is frozen. WP7 must reuse the model and thresholds selected before the shifted evaluation.

---

## 14. Track B promotion criteria

### Agreed principle

Track B is optional and should address a limitation demonstrated by Track A.

A richer neural posterior estimator may be justified if Track A shows one or more of the following:

- rectangular joint regions are materially inefficient because of target dependence;
- marginally calibrated intervals still give poor simultaneous coverage;
- uncertainty appears multimodal and cannot be represented by quantile intervals;
- subgroup coverage differs substantially;
- conformal corrections obtain coverage only by becoming excessively wide;
- the scientific output requires posterior samples rather than intervals;
- the controlled shift exposes a failure that simple interval calibration cannot address.

Diffusion models remain a possible Track B candidate, not a Track A requirement.

---

## 15. Open questions

The following questions remain unresolved and must be answered through WP1 or WP2 without using final test outcomes:

1. What are the exact usable sample counts and feature dimensions in the downloaded dataset?
2. What train-validation-calibration-test proportions provide sufficient calibration and test precision?
3. Which stellar and planetary metadata fields should be included in the baseline input?
4. Is the wavelength grid identical for every example?
5. Is global feature standardisation sufficient, or is spectrum-specific normalisation required?
6. Is a compact multilayer perceptron adequate, or is a small spectral encoder needed?
7. Which single difficulty variable should be used for subgroup diagnostics?
8. Does the subgroup analysis justify a Mondrian extension?
9. What noise-inflation factor should define the main shifted test set?
10. How many repeated seeds are computationally practical for the final analysis?
11. What exact normalised-width or rectangular-region efficiency summary should be used for the final joint comparison?

---

## 16. Assumptions

### Agreed assumptions

- The simulator parameters are available as one target vector per usable Track A example.
- The selected input features are available at prediction time.
- IID conformal claims apply only when calibration and future examples are exchangeable.
- The noise-shift experiment is expected to weaken or break that assumption.
- Coverage alone is insufficient; interval or region efficiency must also be reported.
- Seven separate marginal guarantees do not imply a simultaneous seven-parameter guarantee.
- Track A does not claim to estimate or calibrate the full Bayesian atmospheric posterior.

### Provisional assumptions to verify in WP2

- The available dataset is large enough to reserve distinct validation, calibration and test sets.
- The spectral grid is common across examples.
- A lightweight neural predictor can achieve useful accuracy without challenge-level optimisation.
- Increased observational noise can be constructed consistently from the available spectrum and uncertainty representation.

---

## 17. Definition of Track A completion

Track A is complete when the repository can reproducibly answer the following questions:

- Are the raw quantile intervals calibrated?
- Do split conformal regression and CQR achieve the intended marginal coverage under IID evaluation?
- Which marginal method gives the best coverage-efficiency trade-off?
- What simultaneous coverage results from naive marginal intervals?
- Can direct joint conformal calibration recover the desired simultaneous coverage more efficiently than Bonferroni correction?
- Does global coverage hide a clear subgroup failure?
- How much do coverage and efficiency change under the predefined noise shift?
- Do the results establish a specific reason to proceed to Track B?

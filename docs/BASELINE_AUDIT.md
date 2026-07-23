# ADC2023 Baseline Technical Audit

## Scope

This is a static, read-only technical assessment of the repository. The baseline was not run, and no dependencies or challenge data were installed.

Line references identify the code or documentation supporting each observation. Derived tensor shapes assume the hard-coded 52-channel input described by the repository.

## Observed Facts

### 1. Repository Structure and Entry Points

`run_baseline.py` is the command-line entry point. It accepts `--training` and `--test`, then loads data, preprocesses it, trains the model, predicts the test set, and writes a submission (`run_baseline.py:15-21`, `run_baseline.py:42-51`, `run_baseline.py:100-147`). The entire workflow is at module scope: there is no `main()` function or `if __name__ == "__main__"` guard, so importing the module would also parse arguments and attempt to run the pipeline (`run_baseline.py:15-21`, `run_baseline.py:42-147`).

`ADC2023-baseline.ipynb` is the interactive entry point and additionally demonstrates posterior and spectral validation scoring (`README.md:7-12`, `ADC2023-baseline.ipynb:24-34`, `ADC2023-baseline.ipynb:545-779`).

Supporting modules have the following responsibilities:

- `helper.py` converts spectral HDF5 groups to arrays, standardizes data, augments spectra, and reshapes predictions (`helper.py:4-46`).
- `preprocessing.py` wraps augmentation and standardization (`preprocessing.py:3-17`).
- `MCDropout.py` defines dense and convolutional dropout models (`MCDropout.py:9-38`).
- `posterior_utils.py` supplies prior clipping and posterior scoring (`posterior_utils.py:31-85`).
- `spectral_metric.py` and `FM_utils_final.py` implement spectral scoring and TauREx forward-model utilities (`spectral_metric.py:19-61`, `FM_utils_final.py:86-253`).
- `submit_format.py` writes the competition HDF5 structure (`submit_format.py:7-32`).

The documented usage is direct execution of the script or notebook; the repository does not document a test command, package installation procedure, or reproducible environment setup (`README.md:4-12`, `README.md:25-26`).

### 2. Expected Input Files and Shapes

The training path is expected to contain:

```text
TRAINING/
├── SpectralData.hdf5
├── AuxillaryTable.csv
└── Ground Truth Package/
    ├── FM_Parameter_Table.csv
    └── TraceData.hdf5       # used by notebook validation
```

The command-line loader uses the first three files (`run_baseline.py:43-50`). The notebook additionally opens `TraceData.hdf5` for scoring (`ADC2023-baseline.ipynb:614-624`).

The test path must contain `SpectralData.hdf5` and `AuxillaryTable.csv` (`run_baseline.py:113-115`).

Each spectral HDF5 group is assumed to be named `Planet_<planet_ID>` and contain length-52 datasets named `instrument_wlgrid`, `instrument_spectrum`, `instrument_noise`, and `instrument_width`. These are concatenated, in that order, into an `(N, 52, 4)` matrix (`helper.py:6-19`). The notebook also describes this shape explicitly (`ADC2023-baseline.ipynb:155-168`).

The resulting principal shapes are:

| Data | Shape |
|---|---:|
| Spectral matrix | `(N, 52, 4)` |
| Spectrum or noise matrix | `(N, 52)` |
| Stellar-radius feature | `(N, 1)` |
| Regression targets | `(N, 7)` |
| Augmented spectra | `(repeat * N_train, 52)` |
| Augmented radii | `(repeat * N_train, 1)` |
| Augmented targets | `(repeat * N_train, 7)` |

These shapes follow from channel extraction, the one-column radius selection, seven target columns, and tiling by `repeat` (`run_baseline.py:55-58`, `run_baseline.py:62-75`, `run_baseline.py:80-98`).

`AuxillaryTable.csv` must contain `planet_ID` and `star_radius_m`; notebook spectral evaluation also requires `planet_mass_kg` (`helper.py:7-12`, `run_baseline.py:62-65`, `ADC2023-baseline.ipynb:727-732`). Each posterior group is assumed to expose an `(M, 7)` `tracedata` dataset and a corresponding length-`M` `weights` dataset (`posterior_utils.py:18-27`, `ADC2023-baseline.ipynb:643-654`).

### 3. Preprocessing

Only the first 5,000 selected rows are used (`run_baseline.py:30-32`). The model retains spectral values and noise but discards wavelength centers and widths (`run_baseline.py:54-60`). The notebook states that wavelength grids and widths are unchanged across the dataset, but the implementation does not validate this assumption (`ADC2023-baseline.ipynb:263-267`).

Training spectra are augmented fivefold by adding independent Gaussian draws whose elementwise standard deviations equal the supplied instrumental noise (`helper.py:29-33`, `preprocessing.py:4-7`, `run_baseline.py:83-86`).

All selected spectral values share one global scalar mean and standard deviation. Stellar radius and each target use column-wise statistics (`run_baseline.py:59-60`, `run_baseline.py:67-75`, `run_baseline.py:88-98`).

The only auxiliary feature passed to the model is stellar radius. It is converted from metres using `RSOL = 696340000` and then standardized (`run_baseline.py:28`, `run_baseline.py:62-68`, `run_baseline.py:94-95`). Although the README says that stellar and planetary radii are used as features, the implemented second input contains only stellar radius (`README.md:17-21`, `MCDropout.py:23-31`).

Standardization is a direct `(array - mean) / std` operation. There are no checks for zero variance, missing values, infinities, or malformed rows (`helper.py:23-24`).

### 4. Target Definitions

The output order is:

1. `planet_radius`
2. `planet_temp`
3. `log_H2O`
4. `log_CO2`
5. `log_CO`
6. `log_CH4`
7. `log_NH3`

These seven columns are read from `FM_Parameter_Table.csv` and described in code as soft labels (`run_baseline.py:46-50`, `run_baseline.py:71-75`). They are not derived from the posterior `TraceData.hdf5`, even though that trace is described as the competition ground truth (`README.md:33-35`).

The five gas values are treated as base-10 logarithmic abundances by the forward model (`FM_utils_final.py:147-151`). Radius and temperature are assigned directly to TauREx model parameters (`FM_utils_final.py:142-151`).

### 5. Training, Validation, and Test Handling

A random Boolean mask assigns approximately 80% of the first 5,000 examples to training and 20% to validation (`run_baseline.py:30-32`, `run_baseline.py:78-81`). Splitting occurs before augmentation, and only the training subset is augmented (`run_baseline.py:78-86`).

The model trains for 30 epochs with batch size 32 and reports validation loss. It has no early stopping, checkpointing, learning-rate schedule, or best-weight restoration (`run_baseline.py:103-111`).

Training uses `shuffle=False`. The augmented spectra are flattened from a repeat-first array, while radii and targets are tiled in repeat-sized blocks, so the repeated dataset is consumed in deterministic blocks rather than randomly shuffled (`helper.py:29-33`, `run_baseline.py:84-86`, `run_baseline.py:106-111`).

The command-line script uses test data only for posterior generation and submission; it does not evaluate test labels (`run_baseline.py:113-147`). The notebook computes challenge scores on the validation subset (`ADC2023-baseline.ipynb:545-779`).

### 6. Model Architecture and Output Shapes

`MC_Convtrainer` accepts a spectrum of shape `(batch, 52)` and one auxiliary value of shape `(batch, 1)` (`MCDropout.py:22-25`). It reshapes the spectrum to `(batch, 52, 1)`, then applies three blocks of two kernel-size-3 valid convolutions followed by max pooling (`MCDropout.py:25-30`).

For filters `[32, 64, 64]`, the derived shapes are:

| Stage | Shape excluding batch |
|---|---:|
| Reshape | `(52, 1)` |
| Convolution block 1 | `(24, 32)` |
| Convolution block 2 | `(10, 64)` |
| Convolution block 3 | `(3, 64)` |
| Flatten | `(192,)` |
| Concatenate radius | `(193,)` |
| Dense 1 | `(500,)` |
| Dense 2 | `(100,)` |
| Linear output | `(7,)` |

The filter list is configured in the runner, while the dense head and output are defined in the model module (`run_baseline.py:37`, `MCDropout.py:26-37`). Because convolution padding defaults to `valid`, the architecture depends on a sufficiently long fixed-width input; the loader hard-codes 52 channels (`helper.py:8-19`, `MCDropout.py:27-29`).

### 7. Loss Function and Optimization

The model minimizes mean squared error over seven standardized outputs. It uses Adam with learning rate `1e-3`, batch size 32, and 30 epochs (`run_baseline.py:34-36`, `run_baseline.py:97-111`).

The objective gives each standardized target equal treatment and contains no posterior likelihood, target covariance, physical-prior penalty, or direct challenge-metric term (`run_baseline.py:97-108`). The repository itself notes that joint atmospheric-target structure is not considered (`README.md:35-36`).

### 8. Monte Carlo Dropout

Dropout with rate 0.1 appears only after the 500-unit and 100-unit dense layers; the convolutional blocks contain no dropout (`run_baseline.py:38`, `MCDropout.py:26-36`).

Both dropout layers are constructed with `training=True`, which makes dropout active even during normal validation or calls made with an inference training flag (`MCDropout.py:33-35`). Therefore, the later calls to `model(..., training=True)` are redundant for activating these particular layers (`run_baseline.py:132-135`).

The command-line prediction loop performs 5,000 full-test-set stochastic passes and initially stores an array of shape `(5000, N_test, 7)` (`run_baseline.py:39-40`, `run_baseline.py:129-138`). This distribution reflects masks in two dense layers; test-time observation noise is not resampled, and model weights otherwise remain fixed (`run_baseline.py:118-135`).

### 9. Posterior Prediction and Submission Formatting

Standardized predictions are flattened, inverse-transformed, reshaped to `(samples, planets, 7)`, and axis-swapped to `(planets, samples, 7)` (`helper.py:43-47`, `run_baseline.py:137-141`). Each sample is assigned weight `1 / N_samples`, yielding weights of shape `(N_test, N_samples)` (`run_baseline.py:141-145`).

The writer creates groups named `Planet_public1`, `Planet_public2`, and so on, and gives each a zero-based `ID` attribute (`submit_format.py:18-31`). It assumes rows already appear in ascending competition order and does not accept or preserve the `planet_ID` column (`submit_format.py:7-13`, `submit_format.py:26-31`). This is fragile because loading explicitly follows the auxiliary table's ID sequence (`helper.py:5-12`).

Predictions are not clipped to the stated priors before submission. Prior clipping occurs only within metric code (`posterior_utils.py:46-63`, `spectral_metric.py:48-56`, `run_baseline.py:137-147`).

`to_competition_format` closes the output file but has no return statement, so the runner's `submission` variable receives `None` (`submit_format.py:15-16`, `submit_format.py:18-32`, `run_baseline.py:145-147`).

The notebook initially sets `N_samples = 5000`, later overwrites it with 10 for spectral scoring, and does not restore it before leaderboard generation (`ADC2023-baseline.ipynb:484-490`, `ADC2023-baseline.ipynb:688-692`, `ADC2023-baseline.ipynb:877-902`). Its resulting submission would contain 10 samples, contrary to the notebook's own stated minimum of 1,000 (`ADC2023-baseline.ipynb:856-857`).

### 10. Challenge Metric Implementation

The notebook combines an 80% posterior score with a 20% spectral score (`ADC2023-baseline.ipynb:632-655`, `ADC2023-baseline.ipynb:674-692`, `ADC2023-baseline.ipynb:778-779`).

For the posterior component, both traces are clipped to hard-coded bounds, normalized to `[0, 1]`, equally resampled with `nestle.resample_equal`, and compared target-by-target with the two-sample Kolmogorov-Smirnov statistic. Each target contributes `(1 - KS) * 1000`, and the mean is returned (`posterior_utils.py:31-43`, `posterior_utils.py:46-84`).

The prior bounds have no documented provenance and are accompanied by a source comment saying “check here!!!!!!” (`posterior_utils.py:31-44`). When posterior sample counts differ, `np.resize` repeats or truncates the ground-truth target values rather than statistically resampling to the desired size (`posterior_utils.py:80-84`).

For the spectral component, parameter traces are converted to TauREx spectra and reduced to median and interquartile-width summaries, which are compared in log space (`spectral_metric.py:48-60`, `FM_utils_final.py:78-84`, `FM_utils_final.py:86-117`).

The function named `huber_loss` is not the standard Huber loss: it switches the entire result between MSE when `alpha >= 1` and MAE otherwise (`spectral_metric.py:5-17`). `compute_score` also takes logarithms before checking that spectra and bounds are positive, which can produce invalid values (`spectral_metric.py:31-45`).

`compute_approx_mean_and_bound` equally resamples the trace and then indexes that resampled result using the ordering of the original weights. That ordering no longer has a defined relationship to the resampled rows (`FM_utils_final.py:78-83`).

The notebook passes `idx`, the ordinal validation-loop position, rather than `pl_idx`, the original dataset row, into the dedicated forward-model setup. The radius and mass used for scoring can therefore belong to a different planet (`ADC2023-baseline.ipynb:752-766`, `FM_utils_final.py:215-221`).

Ground-truth traces are skipped only when the number of NaNs is exactly one. A trace containing two or more NaNs is not skipped (`ADC2023-baseline.ipynb:643-654`, `ADC2023-baseline.ipynb:758-767`).

### 11. External Dependencies

Direct third-party dependencies are:

- NumPy (`run_baseline.py:1`)
- TensorFlow/Keras (`run_baseline.py:2-4`, `MCDropout.py:2-7`)
- pandas (`run_baseline.py:3`)
- h5py (`run_baseline.py:5`)
- tqdm (`run_baseline.py:7`)
- SciPy (`posterior_utils.py:72-84`)
- nestle (`posterior_utils.py:65-68`, `FM_utils_final.py:2`)
- matplotlib for visualization (`helper.py:35-41`)
- TauREx 3 for spectral evaluation (`FM_utils_final.py:140-160`, `FM_utils_final.py:173-180`)
- External TauREx opacity, CIA, and line-list data (`README.md:25-26`, `ADC2023-baseline.ipynb:678-717`)

`joblib` is imported but unused (`FM_utils_final.py:55-58`). No versions are pinned in the documented setup; the notebook metadata records Python 3.8.12 (`ADC2023-baseline.ipynb:971-988`).

### 12. Reproducibility Problems

`SEED = 42` is declared near the start of the runner but applied only immediately before test prediction (`run_baseline.py:24-25`, `run_baseline.py:127-134`). It therefore does not seed the train/validation split, Gaussian augmentation, model initialization, training dropout, or validation dropout, all of which occur earlier (`run_baseline.py:78-111`, `helper.py:29-32`).

The notebook likewise assigns the seed early but does not apply it to TensorFlow until leaderboard inference (`ADC2023-baseline.ipynb:67-78`, `ADC2023-baseline.ipynb:861-882`).

No deterministic TensorFlow settings, saved split indices, trained weights, preprocessing statistics, dependency lockfile, or hardware/runtime description are supplied (`run_baseline.py:78-111`, `README.md:4-12`). Metric resampling via `nestle.resample_equal` is also stochastic and receives no explicit random generator or local seed (`posterior_utils.py:65-84`, `FM_utils_final.py:78-83`).

### 13. Possible Data Leakage

The split occurs before augmentation, so augmented copies do not directly cross from training into validation (`run_baseline.py:78-86`). The notebook's final claim that the split happened after augmentation conflicts with the actual ordering of its cells and code (`ADC2023-baseline.ipynb:355-388`, `ADC2023-baseline.ipynb:951-959`).

However, spectral, stellar-radius, and target normalization statistics are all calculated from the full first 5,000 examples before the split (`run_baseline.py:55-75`, `run_baseline.py:78-98`). Validation features affect the input transformations, and validation target values affect target scaling used during training.

Spectra are aligned to auxiliary rows through `planet_ID`, but target rows are selected independently through `soft_label_data.iloc[:N]`; there is no identifier-based join between features and labels (`helper.py:5-19`, `run_baseline.py:49-51`, `run_baseline.py:71-72`). Correct alignment therefore depends on undocumented identical row ordering.

Selecting the first 5,000 rows can introduce selection bias if auxiliary-table order correlates with physical characteristics or label availability (`run_baseline.py:30-32`, `run_baseline.py:55-72`).

### 14. Unclear Assumptions and Missing Documentation

The precise ADC2023 dataset version, schema, units, label semantics, and valid ID conventions are not described beyond filenames and a challenge website link (`README.md:4-12`).

The loader assumes exactly 52 samples in every spectral dataset and a matching HDF5 group for every auxiliary ID (`helper.py:6-19`). It allocates rows using the HDF5 group count but iterates over auxiliary rows, so differing lengths can produce zero-filled trailing rows or an indexing error (`helper.py:6-10`).

CLI path arguments are not marked as required or validated before they are passed to path operations (`run_baseline.py:15-21`, `run_baseline.py:43-49`).

Target units are not documented alongside target loading; they must be inferred from bounds and TauREx assignments (`run_baseline.py:71-75`, `posterior_utils.py:31-44`, `FM_utils_final.py:140-151`).

Submission ordering, HDF5 naming, zero-based `ID` attributes, and accepted sample-count limits are not fully documented in the README (`submit_format.py:7-31`, `ADC2023-baseline.ipynb:856-857`).

Input HDF5 handles are opened without context managers and are not explicitly closed. Only the output submission file is explicitly closed (`run_baseline.py:47-49`, `run_baseline.py:114-115`, `submit_format.py:18-32`).

## Recommendations

### Data Integrity and Leakage

1. Split rows before computing any normalization statistics, and fit spectral, radius, and target scalers using training rows only. This addresses the current use of full-subset statistics before splitting (`run_baseline.py:55-98`).
2. Join spectra, auxiliary data, soft labels, and posterior traces through explicit planet identifiers instead of relying on row order (`helper.py:5-19`, `run_baseline.py:49-72`).
3. Validate HDF5 group counts, required datasets, channel lengths, finite values, and wavelength-grid invariance before allocating the training matrix (`helper.py:6-19`, `ADC2023-baseline.ipynb:263-267`).
4. Replace “first 5000” selection with an explicit, documented labelled-row selection policy or use the complete labelled dataset (`run_baseline.py:30-32`, `README.md:31-34`).

### Reproducibility

1. Apply seeds before the split, augmentation, and model construction, and document deterministic-runtime limitations (`run_baseline.py:24-25`, `run_baseline.py:78-111`, `run_baseline.py:127-134`).
2. Persist split indices, preprocessing statistics, model weights, and configuration alongside each result (`run_baseline.py:78-111`).
3. Add a pinned environment definition covering TensorFlow, SciPy, nestle, TauREx, and the external opacity/CIA data versions (`README.md:25-26`, `FM_utils_final.py:167-183`).
4. Give metric resampling an explicit reproducible random-state policy (`posterior_utils.py:65-84`, `FM_utils_final.py:78-83`).

### Model and Posterior Generation

1. Avoid hard-wiring `training=True` into dropout-layer construction. Control stochastic dropout deliberately at inference and make validation behavior explicit (`MCDropout.py:33-35`, `run_baseline.py:132-135`).
2. Evaluate whether uncertainty should incorporate observational noise at prediction time, model ensembles, or a distributional objective rather than relying only on two dense dropout layers (`MCDropout.py:32-36`, `run_baseline.py:118-135`).
3. Shuffle training rows or document the reason for consuming repeated augmented blocks in fixed order (`run_baseline.py:84-86`, `run_baseline.py:106-111`).
4. Clip or otherwise validate posterior samples against authoritative competition priors before writing a submission (`posterior_utils.py:31-63`, `run_baseline.py:137-147`).

### Metrics

1. Confirm the official prior bounds and cite their source instead of relying on the provisional hard-coded values (`posterior_utils.py:31-44`).
2. Replace `np.resize` with an explicit, statistically valid comparison strategy for unequal posterior sample counts (`posterior_utils.py:80-84`).
3. Implement an actual Huber loss if that is what the challenge metric specifies, and validate positivity before logarithms (`spectral_metric.py:12-17`, `spectral_metric.py:31-45`).
4. Review the resampling and quantile algorithm so ordering and weighting are mathematically consistent (`FM_utils_final.py:78-84`).
5. Pass `pl_idx`, not loop-local `idx`, when selecting the planet-specific radius and mass for spectral validation (`ADC2023-baseline.ipynb:752-766`).
6. Treat any non-finite ground-truth trace as invalid rather than checking for exactly one NaN (`ADC2023-baseline.ipynb:643-654`, `ADC2023-baseline.ipynb:758-767`).

### Submission and Documentation

1. Build submission group names and IDs from explicit test planet identifiers, then validate them against the competition schema (`helper.py:5-12`, `submit_format.py:7-31`).
2. Use distinct variable names for posterior sample count and spectral quantile count so notebook metric setup cannot silently reduce submission samples to 10 (`ADC2023-baseline.ipynb:484-490`, `ADC2023-baseline.ipynb:688-692`, `ADC2023-baseline.ipynb:877-902`).
3. Document file schemas, array shapes, units, target order, ID mapping, sample limits, dependencies, and expected output layout (`README.md:4-26`, `submit_format.py:7-31`).
4. Reconcile documentation with implementation regarding planetary-radius input and train/validation leakage (`README.md:17-21`, `ADC2023-baseline.ipynb:951-959`, `run_baseline.py:62-98`).
5. Use context managers for HDF5 inputs and make the submission writer return the output path or documented result (`run_baseline.py:47-49`, `run_baseline.py:114-115`, `submit_format.py:15-32`).

## Open Questions

1. What exact ADC2023 release and competition schema was this baseline written against, including required HDF5 group names and `ID` attribute conventions (`README.md:1-12`, `submit_format.py:26-31`)?
2. Are `AuxillaryTable.csv` and `FM_Parameter_Table.csv` contractually guaranteed to have identical row order, or should they be joined by planet ID (`helper.py:5-19`, `run_baseline.py:49-72`)?
3. What are the authoritative units and prior bounds for all seven targets (`posterior_utils.py:31-44`, `FM_utils_final.py:140-151`)?
4. Are wavelength centers and widths guaranteed to be identical across every training, validation, and test object (`ADC2023-baseline.ipynb:263-267`, `helper.py:12-19`)?
5. Which examples in the first 5,000 are labelled, and how should unlabelled rows in `FM_Parameter_Table.csv` be identified and excluded (`run_baseline.py:30-32`, `run_baseline.py:71-72`)?
6. Is the supplied posterior score implementation identical to the official evaluator, particularly its prior clipping, equal resampling, and `np.resize` behavior (`posterior_utils.py:46-84`)?
7. Is the supplied spectral score intended to use the implemented MSE/MAE switch, or a standard Huber loss (`spectral_metric.py:12-17`)?
8. Should submission samples outside the prior be rejected, clipped, or left untouched by the official evaluator (`posterior_utils.py:46-63`, `run_baseline.py:137-147`)?
9. Was always-on dropout during validation intentional, or should dropout be activated only for Monte Carlo inference (`MCDropout.py:33-35`)?
10. Should the baseline model a joint seven-dimensional posterior, or are independently induced marginal variations considered sufficient for baseline purposes (`README.md:23-23`, `README.md:35-36`)?

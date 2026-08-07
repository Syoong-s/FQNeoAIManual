# Parameter Tuning Guide: Goal → Exact Knob

Use this file when the user describes a desired behavior rather than naming a symbol. First identify the affected phase/stage, then load the corresponding detailed parameter reference and verify the current source symbol in the selected variant.

## Decision table

| Goal / symptom | First parameters or code to inspect | Scope / warning |
|---|---|---|
| Run or skip a top-level phase | CLI `--run-extcat/init/main/rearr/fd` | Runtime, no rebuild |
| Change science/DQ/output roots or dataset prefix | CLI `--science-root`, `--dq-root`, `--output-root`, `--dataset` | Runtime, no rebuild |
| Change which numerical stages execute | `LensingConfig::PROCESS_stage` | Compile-time; Stage 9 (23) requires Stage 8 (19) |
| Change source/star detection sensitivity | `source_thresh`, `core_thresh`, `area_thresh`, `SNR_PSF`, source-extractor implementation | Compile-time; inspect Stage 3 before changing |
| Change postage-stamp / FFT size | `ns` plus all derived dimensions and I/O assumptions | High-impact compile-time change; inspect stages 3–7 |
| Change PSF model family | Standard: `PSF_type`, `PSF_Ms`; Lite is frozen to local polynomial, no PCA | Do not “unfreeze” Lite by editing one constant |
| Change local PSF fit complexity | `npl`, `nstar_min_local`; inspect `PSFModel` fit basis | Rebuild; ensure enough stars for number of coefficients |
| Enable PCA/multi-scale PSF | Standard `PSF_Ms=1`, PCA parameters | Standard only; requires `PSFRecons.cpp` path |
| Change galaxy/star Fourier smoothing | `gal_smooth`, `star_smooth` | Both currently default to 0/2 respectively; rebuild |
| Change Set_Sig background/noise behavior | `sig_*` constants in `LensingConfig.hpp` + `PreProcess` | Scientific calibration-sensitive; change as a group only with validation |
| Change FQ PSF-radius window | local `PSFr_ratio=0.75f` in `ShearMeasurement::getShear` | Not a config constant |
| Change rearranged subcatalog size | `TARGET_SUBCAT_ROWS` | Compile-time; affects k-d partition count, not sky-grid resolution |
| Change rearrangement sky resolution | `SKY_GRID_DEGREES`, `RA_BIN_COUNT`, `DEC_BIN_COUNT` together | Preserve consistency among the three |
| Change FD spatial bin count/range | `fd_num`, `gf_lim` | Compile-time; affects output and fitting arrays |
| Change FD estimator | `FD_STATIC_MODE` | `PDF_SIGMA`, `PDF_JACK`, `SWSE_JACK` |
| Change FD star-cut model | `FD_PER_EXPOSURE_STAR_BAR`, star histogram/clip parameters | Compile-time; inspect `StarCutCalculator` |
| Change external catalog width/order | CLI explicit projection when possible; otherwise `ExtCatConfig` + downstream layout | Prefer runtime projection for dataset-specific layouts |

## Value-selection protocol

1. Translate the requested outcome into a measurable effect (e.g. more faint detections, fewer PSF stars, smaller subcatalogs, more FD bins).
2. Identify the phase/stage that directly produces that effect; do not tune a downstream threshold to compensate for an upstream defect without evidence.
3. Classify each candidate as **runtime**, **compile-time config**, or **implementation literal/algorithm**.
4. Check coupling constraints and array/layout invariants before changing a value.
5. Prefer one minimal change at a time when the effect is uncertain; preserve a baseline configuration for comparison.
6. Validate with the smallest representative dataset that exercises the changed branch, then scale out.
7. Report the exact file, symbol, old value, new value, expected direction of effect, and whether rebuilding is required.

## Before changing a value

When pipeline source code is available, locate the named symbol in the selected variant and confirm its current definition, direct couplings, and consumers before applying a change.

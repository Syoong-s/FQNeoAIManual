# Changelog

## 1.3.0 - 2026-08-10

- Synchronized Stage 7 with the current C++ pipeline's `gal_size_T` and
  `psf_size_T` low-frequency Fourier-power curvature estimators.
- Documented the single-beta `delta_chi2` and `orth_ext` point-source
  morphology statistics, including their reliable Fourier window, taper,
  formulas, validity sentinels, interpretation, and offline-calibration
  boundary.
- Updated the Stage 7, exposure, combination, rearrangement, and FD catalog
  contracts from 24/25/44 columns to 28/29/48 columns.
- Replaced the removed `npara` description with the live `src_npara`,
  `shear_cat_ncols`, and `expo_cat_ncols` symbols.
- Added the current `test-point-source-statistics` Make target to build and
  developer-validation guidance for both Standard and Lite.

## 1.2.1 - 2026-08-08

- Removed the upstream source-repository snapshot files and routing.
- Removed repository/branch/commit provenance from the skill and README.
- Generalized source-inspection and runner guidance so CFQManual describes Fourier Quad Pipeline Neo (C++) independent of any upstream Git repository.
- Corrected README package paths, skill casing, installation commands, and current plugin version.

## 1.2.0 - 2026-08-08

- Renamed the plugin and skill to `CFQManual`.
- Set author metadata to `Syoong-s` with `https://github.com/Syoong-s`.
- Set the plugin repository/homepage to `https://github.com/Syoong-s/FQNeoAIManual`.
- Added a `v*` tag-triggered GitHub Release workflow that packages `.agents/`, `.claude-plugin/`, `.codex-plugin/`, and `skills/`.
- Updated README installation and release instructions.

## 1.1.0 - 2026-08-07

- Added a dedicated `cpp-10-direct-hpc.md` routing path for lightweight directly runnable HPC Slurm jobs.
- Added direct-HPC templates for an inline single-file Slurm job and an optional small reusable Bash env file.
- Direct mode can acquire/build a shared SIF, generate phase-dependent binds, compile the mounted pipeline when required, build the pipeline CLI, forward Slurm/PMI state under `--cleanenv`, and launch with `srun --mpi=pmi2`.
- Documented that the current runtime image contains the toolchain but not the `cpp_Standard`/`cpp_Lite` source tree, so source binding and executable availability are mandatory.
- Explicitly separated lightweight direct deployment from the repository production runner, preventing unnecessary smoke tests, checksum/image-version checks, and helper validation framework from being copied into simple jobs.

## 1.0.0 - 2026-08-07

- Reworked the uploaded CFQManual into a selective-loading knowledge hierarchy.
- Removed the rule that forced `cpp-01-overview.md` into every task.
- Split the monolithic parameter reference into runtime, main-science, rearr/FD, and tuning-guide references.
- Split the monolithic `process_main` reference into an orchestration router plus nine per-stage detailed references.
- Distinguished `RuntimeOptions`/CLI overrides from build-time scientific constants.
- Corrected current Lite defaults (`F77_MAX_PATH=150`, `gal_smooth=0`).
- Corrected Stage-4 external-PSF routing and Stage-7 `PSFr_ratio` location.
- Documented the current Stage-9 `ext_cat=1` zero-correction branch.
- Removed references to nonexistent current Makefile targets `test-extcat-reader` and `test-rearr`.
- Added source-aware developer modification protocol and Standard/Lite branch-safety rules.
- Added exact runner generation contract plus Docker/HPC environment templates.
- Packaged as a dual-host Codex + Claude plugin using the same manifest/marketplace shape as Superplan, without unnecessary hooks.

# C++ Pipeline: Source-Aware Code Modification Guide

Use this reference for developer changes. Before editing, locate the exact symbol in the selected variant, inspect its declaration/implementation/call sites, and check whether Lite has physically removed the relevant branch.

## File-to-feature map

| Change | Primary location(s) |
|---|---|
| Top-level phase default / RuntimeOptions | `config/ProcessConfig.hpp`, `main.cpp` parser/validation |
| Extcat parsing/projection/defaults | `config/ExtCatConfig.hpp`, `src/process_extcat/`, `src/process_main/ExternalCatalogReader.cpp` |
| Initializer paths/datasets/archive extraction | `config/InitConfig.hpp`, `src/process_init/` |
| Stage schedule / numerical constants | `config/LensingConfig.hpp`, `src/process_main/process_main.cpp` |
| Stage 1 preprocessing/noise | `PreProcess.cpp/.hpp` |
| Stage 2 astrometry/distortion | `Astrometry.cpp/.hpp` |
| Stage 3 source extraction/deblend | `SourceExtractor.cpp/.hpp` |
| Stage 4 star FFT | `FourierTransformSt1.cpp/.hpp` |
| Stage 5 PSF | `PSFModel.cpp/.hpp`; Standard PCA also `PSFRecons.cpp/.hpp` |
| Stage 6 galaxy FFT | `FourierTransformSt2.cpp/.hpp` |
| Stage 7 Fourier_Quad estimator | `ShearMeasurement.cpp/.hpp` |
| Stage 8 exposure diagnostics | `ExposureInfo.cpp/.hpp` |
| Stage 9 combine/calibration | `CatalogCombiner.cpp/.hpp` |
| Rearrangement schema/partitioning | `config/ProcessRearrConfig.hpp`, `src/process_rearr/` |
| FD statistics/star cuts/binning | `config/FDConfig.hpp`, `src/process_fd/` |
| MPI scheduling | `MPIScheduler.cpp/.hpp` |
| FITS I/O | `FitsIO.cpp/.hpp` |
| Linear algebra/numerics | `LinearSolve`, `UniversalUtils`, `NumericalRecipes` |
| Build inputs/libs | variant `Makefile` |
| Docker/HPC runner | `cpp_docker/`, especially `runner/` |

## Required modification protocol

1. **Resolve the variant.** Do not assume a request applies to both Standard and Lite.
2. **Route narrowly.** Load `cpp-06-process-main.md` plus only the affected stage reference, or the relevant phase reference.
3. **Find the live symbol.** Search the checkout for the parameter/function/error text. Do not edit from a remembered line number.
4. **Trace dependency surface.** Read the declaration, implementation, direct call sites, config dependencies, output schema, and tests/smoke path that cover it.
5. **Classify the change:** runtime CLI/default, compile-time config, implementation literal, algorithm, schema, or deployment.
6. **Make the smallest coherent patch.** Preserve current conventions and avoid unrelated refactors unless required.
7. **Handle Standard/Lite deliberately.** Shared behavior may need parallel edits; deleted Lite branches must not be reintroduced accidentally.
8. **Rebuild the affected variant** when source or compile-time config changes.
9. **Validate the branch exercised by the change.** At minimum, clean build and `./Fourier_Quad_Pipe --help`; then use a representative phase/MPI smoke test. The current Makefiles do not provide `test-extcat-reader` or `test-rearr` targets.
10. **Report exact scope:** files/symbols changed, old/new behavior, rebuild requirement, validation performed, and any cross-variant divergence left intentionally.

## Common patterns

### Change a CLI-overridable behavior

Prefer an existing `RuntimeOptions` field and command-line option. If adding a new runtime option:

- add a field/default in the appropriate config/`RuntimeOptions` structure;
- parse both `--name value` and `--name=value` via the unified parser path;
- validate allowed values before MPI work begins;
- propagate the option to the consuming phase without creating a second hidden global default;
- update `printUsage` and this skill's runtime parameter reference.

### Change a compile-time scientific parameter

Edit the exact config symbol in the selected variant and rebuild. Check derived sizes/indices and all uses. For high-coupling values such as `ns`, catalog widths, sky-grid dimensions, or polynomial orders, inspect allocation and file-schema code before changing.

### Change an implementation-local value

Not all knobs are in config. Example: current Stage 7 defines `PSFr_ratio=0.75f` locally in `ShearMeasurement::getShear`. Either change that literal intentionally or promote it into `LensingConfig` as a separate refactor; do not invent a nonexistent config symbol.

### Modify a numerical stage

Use the stage reference to identify the entry point, then trace the actual functions in source. Preserve the stage's input/output contract so adjacent stages continue to read the same products. If that contract changes, update both producer and consumer plus any schema/metadata documentation.

### Add a numerical stage

`PROCESS_stage` uses unique prime factors rather than bit shifts. Adding a stage therefore requires more than inserting a function call: choose a new prime, add scheduling and barriers, define dependency validation, update default product if enabled by default, and verify integer-range/product assumptions. Prefer extending an existing stage unless a new pipeline boundary is genuinely needed.

### Change external-catalog schema

Prefer runtime explicit projection when the change is dataset-specific. For a canonical schema change, inspect together:

- `ExtCatConfig::EXTCAT_TOTAL_COLUMNS` and projection defaults;
- `ExternalCatalogReader` RA/Dec/ZP resolution;
- `FDConfig` external-field indices and `col_ccd` offset;
- `ProcessRearrConfig::externalCatalogColumns/allCatalogColumns`;
- Stage-9 output headers and downstream consumers.

Do not hard-code 44 total columns when explicit projection is enabled; runtime width can differ.

### Change Stage-9 shear calibration

Current default `ext_cat=1` branch in `CatalogCombiner::combineExpoCatalog` sets local `g1c=g2c=0`; the expression using `LensingConfig::g1_c/g2_c` is commented there. The `ext_cat=0` branch uses `gf + g_c`. Modify the active branch that matches the desired catalog mode and validate catalog output numerically.

### Standard vs Lite changes

Lite has deleted eight Standard branches: alternate astrometry, flat/mask branches, no-external-catalog path, external PSF, deblending off, hybrid PSF, and PCA/multi-scale PSF. If a feature depends on one of those, Standard is the native target. Porting it to Lite is a feature restoration, not a parameter tweak.

## Build-system changes

The Makefiles explicitly enumerate `SRCS`. For a new `.cpp`, add it to the appropriate variant's `SRCS`; add include paths/libs only when necessary. If the source is Standard-only (for example a restored PCA feature), do not blindly add it to Lite.

The current Makefile has only `all` and `clean`. If you add tests, choose an explicit test build/run mechanism and document it; do not assume historical test target names exist.

## Debugging orientation

- **Stage not running:** factor `PROCESS_stage` by the stage's prime; Stage 9 without Stage 8 is rejected explicitly.
- **Column/layout errors:** compare runtime projection width with `ExternalCatalogReader`, `ProcessRearrConfig`, FD offsets, and actual catalog headers.
- **MPI failures:** distinguish manager-worker stage work from collective calls; verify all ranks take compatible collective paths.
- **Path failures:** distinguish host bind paths, container paths, runtime CLI roots, and compile-time paths.
- **Lite compile errors after a Standard patch:** first check whether the patch referenced a branch/file removed from Lite.

## Update this skill when code changes

For a behavior-changing pipeline patch, update the smallest corresponding detailed reference and, if routing changed, `SKILL.md`. Avoid copying full source code into the skill; document exact symbols, contracts, invariants, and decision logic instead.

# Standard vs Lite: Variant Differences

## Overview

The C++ pipeline has two build variants that share the same directory layout
and CLI interface but differ in their compile-time branch switches:

- **`cpp_Standard`** - full build with all eight compile-time branches and PCA
  PSF reconstruction (`PSFRecons`)
- **`cpp_Lite`** - simplified build with eight branches frozen to single values
  and `PSFRecons` removed

Both produce the same executable name (`Fourier_Quad_Pipe`) and accept the same
command-line options.

## Frozen Branch Switches (Lite)

Lite fixes eight compile-time switches to their current values and **physically
deletes** all unselected branch code and corresponding `if` statements:

| Switch | Frozen value | Preserved behavior |
|:---|:---:|:---|
| `ASTROMETRY_trivial` | `0` | Gaia-based astrometry only |
| `include_FLAT` | `0` | No super-flat multiplication |
| `include_Mask` | `2` | Per-chip DQ mask from `dirOutput/dqmask/<expo>_<cid>.fits` |
| `ext_cat` | `1` | Use external source catalog `SOURCE_CAT` |
| `ext_PSF` | `0` | PSF measured from frame stars |
| `deblending` | `1` | Deblending always enabled |
| `PSF_type` | `1` | Local polynomial PSF fit only |
| `PSF_Ms` | `0` | No multi-scale / PCA PSF reconstruction |

## Retained Optional Branches (Lite)

These remain configurable in Lite's `LensingConfig.hpp`:
- `PROCESS_stage` (stage bitmask)
- `CCD_split` (1 or 2)
- `gal_smooth` (currently 0 in both variants)
- `star_smooth`

## Parameter Differences

| Parameter | Standard | Lite | Location |
|:---|:---:|:---:|:---|
| `RUN_PROCESS_REARR` | `true` | `false` | `ProcessConfig.hpp` |
| `RUN_PROCESS_FD` | `true` | `false` | `ProcessConfig.hpp` |
| `ASTROMETRY_trivial` | `0` (selectable) | absent (frozen to 0) | `LensingConfig.hpp` |
| `include_FLAT` | `0` (selectable) | absent (frozen to 0) | `LensingConfig.hpp` |
| `include_Mask` | `2` (selectable) | absent (frozen to 2) | `LensingConfig.hpp` |
| `ext_cat` | `1` (selectable) | absent (frozen to 1) | `LensingConfig.hpp` |
| `ext_PSF` | `0` (selectable) | absent (frozen to 0) | `LensingConfig.hpp` |
| `PSF_type` | `1` (selectable) | absent (frozen to 1) | `LensingConfig.hpp` |
| `PSF_Ms` | `0` (selectable) | absent (frozen to 0) | `LensingConfig.hpp` |
| `FLAT_PATH` | present | absent | `LensingConfig.hpp` |
| `PSF_PATH` | present | absent | `LensingConfig.hpp` |
| PCA params (`n_pcs`, etc.) | present | absent | `LensingConfig.hpp` |

**Note:** The default dataset (`gband:c4d_`), `F77_MAX_PATH` (`150`), and
`gal_smooth` (`0`) are currently **identical** in both variants. Treat any older parameter table that says Lite `F77_MAX_PATH=149` or `gal_smooth=2` as stale; current `main` has `F77_MAX_PATH=150` and `gal_smooth=0` in Lite.

## Per-File Behavioral Changes (Lite vs Standard)

The following differences are structural; exact line counts are intentionally omitted because they are brittle across commits.

| File/module | Lite change relative to Standard |
|:---|:---|
| `LensingConfig.hpp` | Removes the eight frozen switches plus paths/parameters used only by deleted branches, including PCA controls |
| `MPIScheduler.*` | Removes PCA-only `forcecov()` support |
| `Makefile` | Omits `PSFRecons.cpp` |
| `main.cpp` / `process_main.cpp` | Removes PCA reconstruction setup/cleanup calls where applicable |
| `PreProcess.*` | Flattens selected mask/flat/astrometry branches |
| `Astrometry.*` | Keeps Gaia-based path only |
| `SourceExtractor.*` | Flattens the external-catalog/deblending-selected behavior |
| `PSFModel.*` | Keeps local polynomial PSF path; removes hybrid/PCA helpers |
| `PSFRecons.cpp/.hpp` | Deleted |
| `ShearMeasurement.cpp` | Removes external-PSF/PCA/hybrid branches tied to frozen switches |
| `FourierTransformSt1.cpp` | Removes the Standard external-PSF early-return branch because Lite cannot enable external PSF |
| `CatalogCombiner.cpp` | Flattens the `ext_cat=1` path and removes the alternate no-external-catalog branch |

## Unchanged Files

These are byte-for-byte copies between variants:
- `FitsIO.*`
- `ImageProcessing.*`
- `LinearSolve.*`
- `NumericalRecipes.*`
- `UniversalUtils.*`
- `ExStar.*`
- `ExposureInfo.*`
- `FourierTransformSt2.*`
- `FourierTransformSt1.hpp`
- `CatalogCombiner.hpp`
- `ShearMeasurement.hpp`

## Deliberately Retained Unused Code

`LinearSolve.cpp/.hpp`, `UniversalUtils.cpp/.hpp`, and `FitsIO.cpp/.hpp` are
unchanged even though some functions lost their callers when `PSFRecons` was
deleted. These are general-purpose utility libraries (like `FFTPACK.f` /
`press.f` in the Fortran tree) and are retained wholesale.

Functions that lost callers (candidates for cleanup):

| Function | Former caller |
|:---|:---|
| `LinearSolve::analyzeCovarianceSpectrum` | `PSFRecons::chipResPCAFit` |
| `UniversalUtils::fit2D2` | `PSFRecons::itpNormPSFCov` |

## When to Choose Which

### Use `cpp_Standard` when:
- You need PCA PSF reconstruction (`PSF_Ms=1`)
- You need external PSF images (`ext_PSF=1`)
- You need super-flat correction (`include_FLAT=1`)
- You need trivial astrometry (`ASTROMETRY_trivial=1`)
- You need legacy mask modes (`include_Mask=1` or `3`)
- You need hybrid PSF (`PSF_type=2`)
- You are developing or testing optional branches

### Use `cpp_Lite` when:
- You are running the standard DES Y6 processing configuration
- You want a simpler, smaller codebase
- You want faster compilation
- You don't need any of the optional branches above
- You want the frozen-branch guarantee (no accidental branch selection)

## Migration Between Variants

When moving a config header between variants:
- **Do NOT** copy one `LensingConfig.hpp` over the other
- Preserve only differences that actually exist in the live checkout; current default dataset, `F77_MAX_PATH=150`, and `gal_smooth=0` are the same in both variants
- Lite's `LensingConfig.hpp` has fewer parameters (the deleted ones are absent)
- The `PSF_Ms=1` code path and `PSFRecons.cpp` do not exist in Lite

## Compilation verification

`cpp_Lite/REFACTOR_NOTES.md` records historical compile/link verification for the refactor. Treat those recorded warning/symbol counts as historical refactor evidence, not as a permanent invariant. For a developer change, rebuild the current selected variant and validate the affected path rather than comparing against old line/warning counts.

## Developer rule for Lite

Lite is not merely Standard with different constant values. Its eight frozen branches were physically deleted. If a requested feature depends on a deleted branch (external PSF, PCA PSF, alternate mask/flat/astrometry paths, etc.), modify Standard or deliberately port the full branch and dependencies into Lite; do not add a missing constant and assume the implementation exists.

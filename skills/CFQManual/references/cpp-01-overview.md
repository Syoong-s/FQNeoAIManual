# C++ Pipeline: Architecture & Execution Model

## Overview

The C++17 Fourier_Quad pipeline is an MPI-parallel weak-lensing shear
measurement system that processes DES (Dark Energy Survey) DECam exposures from
raw archive images to calibrated shear catalogs. It is the C++ successor to the Fortran 77 pipeline and preserves the same Fourier_Quad processing lineage, while reorganizing the implementation into a modular, namespace-based C++17 codebase with a unified command-line interface. Do not assume byte-for-byte or branch-for-branch equivalence when debugging a numerical difference; inspect the current C++ and legacy implementations involved.

Two build variants share the same source layout and CLI:

- **`cpp_Standard`** - full build with PCA PSF reconstruction (`PSFRecons`)
  and all eight compile-time branch switches
- **`cpp_Lite`** - simplified build with eight branches frozen to single
  values and `PSFRecons` removed (see
  [cpp-09-standard-vs-lite.md](cpp-09-standard-vs-lite.md))

## Five-Function Architecture

The pipeline driver (`main.cpp`) orchestrates five top-level functions in a
fixed order. Each is gated by a boolean phase switch and runs on the same MPI
communicator (`MPI_COMM_WORLD`):

```
┌─────────────────────────────────────────────────────────────────────┐
│                        main.cpp driver                               │
│                                                                      │
│  ┌──────────────┐  ┌─────────────┐  ┌──────────────┐               │
│  │ process_extcat│->│ process_init│->│ process_main │               │
│  │  (once)       │  │ (per dataset)│  │ (per dataset) │               │
│  └──────────────┘  └─────────────┘  └──────┬───────┘               │
│                                            │                         │
│                     ┌──────────────────────┘                         │
│                     ▼                                                │
│              ┌───────────────┐  ┌───────────┐                       │
│              │ process_rearr │->│process_fd │                       │
│              │ (per dataset) │  │(per dataset)│                     │
│              └───────────────┘  └───────────┘                       │
└─────────────────────────────────────────────────────────────────────┘
```

| # | Function | Phase switch | Scope | Purpose |
|:---:|:---|:---|:---|:---|
| 1 | `process_extcat` | `--run-extcat` | once, before datasets | Split raw external catalogs into 0.1° sky tiles |
| 2 | `process_init` | `--run-init` | per dataset | Extract Science/DQ chips, build directory tree, publish expo lists |
| 3 | `process_main` | `--run-main` | per dataset | Run 9-stage numerical shear pipeline |
| 4 | `process_rearr` | `--run-rearr` | per dataset | Rearrange `_all.cat` into spatial subcatalogs |
| 5 | `process_fd` | `--run-fd` | per dataset | FD shear test: recover mean shear per field-distortion bin |

**Rules:**
- At least one phase must be enabled (all-`false` is rejected).
- `process_extcat` runs once before the dataset loop; all others run per
  dataset in the order init -> main -> rearr -> fd.
- An enabled `process_rearr` always follows an enabled `process_main` (if both
  are on). An enabled `process_fd` follows `process_rearr` (if both are on).
- Datasets execute **sequentially** on the same communicator; the pipeline
  stops at the first failure.
- `process_init` failure is collective; the numerical phase is never entered
  after initialization fails.

## Source Directory Layout

Each variant (`cpp_Standard/` or `cpp_Lite/`) has this structure:

```
cpp_Standard/
├── main.cpp                    # MPI entry point, CLI parsing, 5-function orchestration
├── Makefile                    # Build file (mpicxx, C++17, CFITSIO+FFTW+LAPACK)
├── README.md
├── config/                     # All compile-time configuration headers
│   ├── ProcessConfig.hpp       #   Workflow defaults + RuntimeOptions struct
│   ├── ExtCatConfig.hpp        #   External-catalog tiling defaults
│   ├── InitConfig.hpp          #   Initializer + exposure-list defaults
│   ├── LensingConfig.hpp       #   Scientific/numerical constants
│   ├── ProcessRearrConfig.hpp  #   Rearrangement parameters + derived column layout
│   └── FDConfig.hpp            #   FD shear-test parameters
├── include/                    # Headers (mirrors src/ structure)
│   ├── process_extcat/
│   ├── process_init/
│   ├── process_main/           #   All stage + support module headers
│   ├── process_rearr/
│   └── process_fd/             #   FD test module headers
└── src/                        # Implementation files
    ├── process_extcat/
    ├── process_init/
    ├── process_main/           #   Stage 1-9 implementations + support modules
    ├── process_rearr/
    └── process_fd/             #   FD test module implementations
```

### Key files by function

| Function | Orchestration | Config | Source modules |
|:---|:---|:---|:---|
| `process_extcat` | `main.cpp` | `ExtCatConfig.hpp` | `src/process_extcat/process_extcat.cpp` |
| `process_init` | `main.cpp` | `InitConfig.hpp` | `src/process_init/{process_init,Initializer,FitsExtractor}.cpp` |
| `process_main` | `src/process_main/process_main.cpp` | `LensingConfig.hpp` | `src/process_main/{PreProcess,Astrometry,SourceExtractor,FourierTransformSt1,PSFModel,PSFRecons,FourierTransformSt2,ShearMeasurement,ExposureInfo,CatalogCombiner,...}.cpp` |
| `process_rearr` | `main.cpp` | `ProcessRearrConfig.hpp` | `src/process_rearr/{process_rearr,CatalogRearranger}.cpp` |
| `process_fd` | `main.cpp` | `FDConfig.hpp` | `src/process_fd/{process_fd,ShearCatalogReader,StarCutCalculator,KMeansClusterer,FDMeasurement,QuadraticFitting}.cpp` |

## MPI Execution Model

### Master-worker dynamic load balancing

All exposure-level work uses `MPIScheduler::distribute()` - a manager-worker
model:

- Rank 0 acts as the **manager**, distributing exposure indices (1-based) to
  worker ranks.
- Each worker requests the next exposure, processes it, and requests another.
- This provides automatic load balancing: faster nodes process more exposures.
- A barrier (`MPIScheduler::barrier()`) separates stages.

### Collective operations

- **Broadcast**: Rank 0 loads the exposure list and broadcasts it to all ranks
  via `MPI_Bcast` (count, then length-prefixed strings).
- **Reduction**: Stage 8 (ExposureInfo) reduces per-exposure diagnostics across
  all ranks via `MPI_Allreduce` (sum), then rank 0 writes `expo_info.dat`.
- **Column validation**: `process_main` validates external-catalog column
  configuration collectively (`MPI_Allreduce` with `MPI_MIN`).

### Per-chip sequential processing

Within each exposure, up to 62 CCD chips (`NMAX_CHIP = 62`) are processed
sequentially. The pipeline reads the per-exposure chip list file and iterates
over chip image paths.

## Data Flow

```
Raw .fits.fz archives
    │
    ▼ process_init
OUTPUT/TARGET/
├── science/<EXPO>/<EXPO>_<N>.fits    (Science chip images)
├── dqmask/<EXPO>/<EXPO>_<CCDNUM>.fits (DQ mask chips)
├── stamps/<EXPO>.list                 (per-exposure chip lists)
└── expo_TARGET.list, fits_TARGET.list, init_TARGET_manifest.json
    │
    ▼ process_main (9 stages)
stamps/
├── Norm/, cat_Orig/, dat_StarInfo/, dat_StarCanInfo/, dat_SrcInfo/
├── dat_PsfFit/, dat_Shear/, dat_ExpoInfo/, dat_StarComp[V2]/
├── dat_Rescale/, dat_StarXY/, dat_Pcs/
├── fits_StarCan[N/P]/, fits_StarP/, fits_Src[P]/, fits_Noise/
├── fits_PsfLocal/, fits_PsfSrc/, fits_PsfResi/
└── astrometry/{dat_Astro,Head,dat_Chk}/
    │
    ▼ Stage 9 (CatalogCombiner)
result/<EXPO>_all.cat                   (per-exposure shear catalogs)
    │
    ▼ process_rearr
<resolved rearr output>/   (default: <dataset_root>/baked/)
├── subcat_NNNNNN.cat                   (spatially sorted subcatalogs)
└── catalog_summary.txt                  (partition count + RA/Dec bounds)
    │
    ▼ process_fd
fdout/FD_test_comb.dat                  (per-bin shear recovery results)
```

## Stage Scheduling (Prime-Factor Bitmask)

An integer `PROCESS_stage` (in `LensingConfig.hpp`) acts as a bitmask via
prime factor decomposition. Each stage corresponds to a unique prime:

| Stage | Prime | Function | Core Algorithm |
|:---:|:---:|:---|:---|
| 1 | 2 | Pre-processing | Background removal, noise normalization, Gaia matching |
| 2 | 3 | Astrometry | PU polynomial sky-coordinate mapping |
| 3 | 5 | Source detection | SNR threshold detection, deblending, star candidates |
| 4 | 7 | FFT-1 | Power spectrum for star candidates |
| 5 | 11 | PSF modeling | χ² star selection, spatial polynomial (+ PCA if `PSF_Ms=1`) |
| 6 | 13 | FFT-2 | Power spectrum for galaxies |
| 7 | 17 | Shear measurement | Fourier_Quad shear estimation (5 estimators) |
| 8 | 19 | Exposure info | Chip -> exposure diagnostics aggregation |
| 9 | 23 | Catalog combine | Quality cuts, distortion calibration, merged catalog |

A stage executes when `PROCESS_stage % prime == 0`. The default value is the
product of all 9 primes, enabling end-to-end processing. **Stage 9 requires
Stage 8** (the pipeline rejects `PROCESS_stage` with factor 23 but without 19).

For detailed stage implementation, see
[cpp-06-process-main.md](cpp-06-process-main.md). For the scientific algorithm
(formulas, derivations), see the `fqmethod` skill.

## Key Global Parameters

| Parameter | Value | Location | Meaning |
|:---|:---:|:---|:---|
| `PROCESS_stage` | 223092870 | `LensingConfig.hpp` | Full pipeline stage switch |
| `ns` | 64 | `LensingConfig.hpp` | Stamp / power spectrum size (px) |
| `npx` × `npy` | 3000 × 5000 | `LensingConfig.hpp` | Nominal CCD size (px) |
| `NMAX_EXPO` | 25000 | `LensingConfig.hpp` | Max exposures per run |
| `NMAX_CHIP` | 62 | `LensingConfig.hpp` | Max CCDs per exposure |
| `npara` | 25 | `LensingConfig.hpp` | Shear catalog fields per source |
| `EXTCAT_TOTAL_COLUMNS` | 18 | `ExtCatConfig.hpp` | Canonical external catalog width |
| `SKY_GRID_DEGREES` | 0.1 | `ProcessRearrConfig.hpp` | Rearrangement sky-tile width (°) |
| `fd_num` | 21 | `FDConfig.hpp` | FD test spatial bins |
| `pixel_size` | 0.2628 | `LensingConfig.hpp` | DECam pixel scale (arcsec) |

## Configuration Model: Runtime Overrides vs Build-Time Constants

Do not treat the config headers as one global precedence stack. The driver seeds `ProcessConfig::RuntimeOptions` from compiled defaults in `ProcessConfig`, `ExtCatConfig`, and `InitConfig`, then CLI parsing overrides the fields represented by that struct. Those changes apply to one invocation without rebuilding.

`LensingConfig.hpp`, most of `FDConfig.hpp`, and other symbols not copied into `RuntimeOptions` remain compile-time controls. Changing them requires rebuilding the selected variant. Some values are implementation-local rather than config symbols (for example the current Stage-7 `PSFr_ratio=0.75f`).

Use [cpp-03-parameters.md](cpp-03-parameters.md) as the parameter router and verify the exact symbol in the live source tree before editing.

For build commands and CLI syntax, see [cpp-02-build-and-run.md](cpp-02-build-and-run.md).

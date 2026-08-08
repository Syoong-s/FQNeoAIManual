# process_main: The 9-Stage Numerical Pipeline

## Purpose

`process_main` executes the Fourier_Quad shear measurement pipeline: 9 stages
that transform raw DECam chip images into per-exposure shear catalogs
(`_all.cat`). It is the scientific core of the pipeline.

## Orchestration

**Entry point:** `src/process_main/process_main.cpp`

```cpp
int process_main(const std::string& exposure_list,
                 const ProcessConfig::RuntimeOptions& options);
```

### Flow

1. **Configure external-catalog columns** (collective validation via
   `ExternalCatalogReader::configure` + `MPI_Allreduce`)
2. **Load exposure list** (rank 0 reads, validates, broadcasts to all ranks)
3. **Validate stage dependency** (Stage 9 requires Stage 8)
4. **Execute stages** in order, each via `MPIScheduler::distribute(N_EXPO,
   stageFunction, message)` with a barrier between stages
5. **Stage 8 reduction** (`MPI_Allreduce` sum of exposure diagnostics, rank 0
   writes `expo_info.dat`)
6. **Stage 9** (catalog combination, per-exposure)

### Stage scheduling

Each stage is gated by `PROCESS_stage % prime == 0`. The stage functions are:

```cpp
if (PROCESS_stage % 2  == 0) MPIScheduler::distribute(N_EXPO, PreProcess::preProcess, ...);
if (PROCESS_stage % 3  == 0) MPIScheduler::distribute(N_EXPO, Astrometry::procAstrometry, ...);
if (PROCESS_stage % 5  == 0) MPIScheduler::distribute(N_EXPO, SourceExtractor::procSource, ...);
if (PROCESS_stage % 7  == 0) MPIScheduler::distribute(N_EXPO, FourierTransformSt1::procFourierTSt1, ...);
if (PROCESS_stage % 11 == 0) {
    MPIScheduler::distribute(N_EXPO, PSFModel::procPSF, ...);
    if (PSF_Ms == 1) PSFRecons::chipPSFRecons(N_EXPO);   // Standard only
}
if (PROCESS_stage % 13 == 0) MPIScheduler::distribute(N_EXPO, FourierTransformSt2::procFourierTSt2, ...);
if (PROCESS_stage % 17 == 0) MPIScheduler::distribute(N_EXPO, ShearMeasurement::procShear, ...);
if (PROCESS_stage % 19 == 0) MPIScheduler::distribute(N_EXPO, ExposureInfo::procInfo, ...);
// MPI_Allreduce of expo_para -> write expo_info.dat
if (PROCESS_stage % 23 == 0) MPIScheduler::distribute(N_EXPO, CatalogCombiner::procComb, ...);
```

### Stage dependency rule

Stage 9 (`prime 23`) requires Stage 8 (`prime 19`). The pipeline rejects
`PROCESS_stage` with factor 23 but without 19:
```
Error: Stage 9 requires Stage 8. PROCESS_stage enables CatalogCombiner without ExposureInfo.
```

---

## Load only the stage you need

| Stage | Prime | Detailed reference |
|---:|---:|---|
| 1 Pre-processing | 2 | [cpp-06-stage-01-preprocess.md](cpp-06-stage-01-preprocess.md) |
| 2 Astrometry | 3 | [cpp-06-stage-02-astrometry.md](cpp-06-stage-02-astrometry.md) |
| 3 Source detection | 5 | [cpp-06-stage-03-source.md](cpp-06-stage-03-source.md) |
| 4 FFT stars | 7 | [cpp-06-stage-04-fft-stars.md](cpp-06-stage-04-fft-stars.md) |
| 5 PSF model | 11 | [cpp-06-stage-05-psf.md](cpp-06-stage-05-psf.md) |
| 6 FFT galaxies | 13 | [cpp-06-stage-06-fft-galaxies.md](cpp-06-stage-06-fft-galaxies.md) |
| 7 Shear measurement | 17 | [cpp-06-stage-07-shear.md](cpp-06-stage-07-shear.md) |
| 8 Exposure info | 19 | [cpp-06-stage-08-exposure.md](cpp-06-stage-08-exposure.md) |
| 9 Catalog combine | 23 | [cpp-06-stage-09-combine.md](cpp-06-stage-09-combine.md) |

For a code change, load this router plus **only the affected stage file** and [cpp-11-code-modification.md](cpp-11-code-modification.md).

## Support Modules

| Module | Files | Purpose |
|:---|:---|:---|
| `MPIScheduler` | `MPIScheduler.cpp/.hpp` | MPI init/finalize, master-worker distribution, barrier |
| `OutputFile` | `OutputFile.hpp` | Checked text output; reports path/reason and aborts the MPI job on failure |
| `NumericalRecipes` | `NumericalRecipes.cpp/.hpp` | RNG (`ran1`), sorting, interpolation (ported from NR) |
| `FitsIO` | `FitsIO.cpp/.hpp` | CFITSIO wrappers for FITS image/catalog I/O |
| `LinearSolve` | `LinearSolve.cpp/.hpp` | Linear algebra, eigen-decomposition (LAPACK), covariance spectrum |
| `ImageProcessing` | `ImageProcessing.cpp/.hpp` | Image manipulation utilities |
| `UniversalUtils` | `UniversalUtils.cpp/.hpp` | Shared utilities (path, polynomial fitting, catalog) |
| `ExStar` | `ExStar.cpp/.hpp` | Star extraction and classification |
| `ExternalCatalogReader` | `ExternalCatalogReader.cpp/.hpp` | External catalog column resolution and reading |
| `PSFRecons` | `PSFRecons.cpp/.hpp` | PCA PSF reconstruction (Standard only, `PSF_Ms=1`) |

### Output failure and path-length contract

All text outputs created by `process_main` use `MainIO::OutputFile`; all FITS
outputs pass through checked `FitsIO` create/write/close operations. Failure
prints `Output creation failed` with rank, operation, path, and the OS/CFITSIO
reason, then calls `MPI_Abort`. A worker must not merely return or call local
`exit`, because the dynamic scheduler can otherwise strand other ranks in a
receive or barrier.

The Stage 1--9 producer/consumer path chain has been checked end to end. Chip
products use `OutputLayout::chipPath` on both sides; exposure products remain
flat in their type directory; DQ reads match initializer output at
`dqmask/<EXPOSURE>/<EXPOSURE>_<CCDNUM>.fits`. The checked sequence is
astro/norm -> head/check -> source and star candidates -> star power -> PSF
products -> source power -> shear -> exposure info -> `_all.cat`; no current
layer or suffix mismatch was found.

`process_main` has no fixed 150-character `strl` limit. Its paths use
`std::string` and are bounded by the filesystem and underlying I/O library.
The default 150-character `--f77-max-path` check belongs only to
`process_init` and can be disabled with `0`.

## Intermediate Products (under `stamps/`)

Type-specific subdirectories:

Chip-scoped products are sharded one additional level by exposure:
`<type-directory>/<EXPOSURE>/<CHIP><suffix>`. This applies to `Norm`,
`cat_Orig`, `dat_StarCanInfo`, `fits_StarCan`, `fits_StarCanN`,
`fits_StarCanP`, `dat_SrcInfo`, `fits_Src`, `fits_Noise`, `fits_SrcP`,
`dat_PsfFit`, `fits_PsfLocal`, `dat_Shear`, `dat_StarXY`, and
`fits_PsfResi`; chip astrometry likewise uses
`astrometry/dat_Astro/<EXPOSURE>/`. Exposure-scoped and CCD/PCA-global
products remain directly in their type directories.

| Directory | Content |
|:---|:---|
| `Norm/` | Normalized images |
| `cat_Orig/` | Original source catalogs |
| `dat_StarInfo/` | Star information |
| `dat_StarCanInfo/` | Star candidate info |
| `dat_SrcInfo/` | Source info |
| `dat_PsfFit/` | PSF fit data |
| `dat_Shear/` | Shear measurement data |
| `dat_ExpoInfo/` | Exposure info |
| `dat_StarComp/`, `dat_StarCompV2/` | Star comparison data |
| `dat_Rescale/` | Rescale factors |
| `dat_StarXY/` | Star positions |
| `dat_Pcs/` | PCA data |
| `fits_StarCan/`, `fits_StarCanN/`, `fits_StarCanP/` | Star candidate FITS |
| `fits_StarP/` | Star power spectra |
| `fits_Src/`, `fits_SrcP/` | Source/galaxy power spectra |
| `fits_Noise/` | Noise power spectra |
| `fits_PsfLocal/`, `fits_PsfSrc/`, `fits_PsfResi/` | PSF models |
| `astrometry/dat_Astro/`, `astrometry/Head/`, `astrometry/dat_Chk/` | Astrometry solutions, WCS, check data |

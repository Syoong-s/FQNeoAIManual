# C++ Pipeline: Build & Run Guide

## Toolchain Requirements

| Component | Local verification | Portable HPC target |
|:---|:---|:---|
| G++ | 15.2.0 | 12.3.0 |
| MPI C++ wrapper | `mpicxx` (Open MPI 5.0.10) | `mpicxx` (Open MPI 4.1.8, PMI2) |
| CFITSIO | 4.6.3 | 4.6.4 |
| FFTW | 3.3.10 | 3.3.11 |
| Eigen | 3.4.0 | 3.4.0 |
| LAPACK / BLAS | OpenBLAS 0.3.33 | LAPACK 3.11.0 / OpenBLAS 0.3.33 |

A cluster may use equivalent site modules as long as one MPI C++ wrapper
compiles and launches the program. No Windows-native compiler is required; the
pipeline runs on Linux in WSL2.

## Makefile

The `Makefile` (in each variant root: `cpp_Standard/` or `cpp_Lite/`) uses:

```makefile
CXX = mpicxx
CPPFLAGS = -Iinclude -Iconfig -Iinclude/process_extcat -Iinclude/process_init \
           -Iinclude/process_main -Iinclude/process_rearr -Iinclude/process_fd
CXXFLAGS = -O3 -std=c++17 -Wall -Wextra -MMD -MP
LDLIBS = -lcfitsio -lfftw3 -lfftw3f -llapack -lblas -lm
TARGET = Fourier_Quad_Pipe
```

### Build commands

```bash
# Standard build (all headers/libs in default search paths)
make -j4

# Consolidated scientific-stack prefix (headers + Eigen + libs under one root)
make STACK_PREFIX=/opt/science-stack -j4

# Separate Eigen and library prefixes
make CXX=/path/to/mpicxx \
  STACK_PREFIX=/opt/libs \
  EIGEN_INCLUDE=/opt/eigen/include/eigen3 -j4
```

### Variables

| Variable | Default | Purpose |
|:---|:---|:---|
| `CXX` | `mpicxx` | MPI C++ compiler wrapper |
| `STACK_PREFIX` | (empty) | Consolidated prefix for `-I`/`-L`/`-rpath` |
| `EIGEN_INCLUDE` | (empty) | Separate Eigen include path |
| `CXXFLAGS` | `-O3 -std=c++17 -Wall -Wextra -MMD -MP` | Compiler flags |
| `LDLIBS` | `-lcfitsio -lfftw3 -lfftw3f -llapack -lblas -lm` | Link libraries |

### Validation targets

The current variant Makefiles expose `all`, `clean`, and the focused Stage 7
`test-point-source-statistics` target. They do **not** define
`test-extcat-reader` or `test-rearr` Make targets. After a clean build, use the
executable's parser as a low-cost validation and then run a representative
phase/smoke test appropriate to the change:

```bash
make clean && make -j4
./Fourier_Quad_Pipe --help
```

For Stage 7 point-source morphology or catalog-width changes, run in the
affected variant:

```bash
make test-point-source-statistics
```

The target builds `tests/PointSourceStatisticsTest.cpp` with the production
module and validates synthetic PSF-like/extended inputs plus the 28/29-column
contract. If another standalone test exists, invoke it by its actual current
instructions rather than assuming a Make target exists.

### Clean

```bash
make clean    # Remove objects, deps, and executable
```

## CLI Option Syntax

The driver (`main.cpp`) accepts options in any order:

- **`--name value`** and **`--name=value`** are both accepted.
- Boolean values accept: `true`, `false`, `1`, `0`, `on`, `off`.
- Duplicate scalar options use the **last** value.
- The first explicit `--dataset`, `--contains`, or `--extcat-contains` **clears**
  its configured list; later occurrences **append**.
- A single bare positional argument is accepted as a legacy exposure-list alias
  (equivalent to `--expo-list`).
- `--help` prints configured dataset defaults.

## Phase Switches

| CLI option | Config default (Std) | Purpose |
|:---|:---:|:---|
| `--run-extcat BOOL` | `false` | Enable external-catalog repartitioning |
| `--run-init BOOL` | `true` | Enable archive initialization |
| `--run-main BOOL` | `true` | Enable the 9-stage numerical pipeline |
| `--run-rearr BOOL` | `true` | Enable `_all.cat` spatial rearrangement |
| `--run-fd BOOL` | `true` | Enable FD shear test |

At least one must be `true`. These override the `RUN_PROCESS_*` constants in
`ProcessConfig.hpp` without rebuilding.

> **Lite difference:** `cpp_Lite` defaults `RUN_PROCESS_REARR=false` and
> `RUN_PROCESS_FD=false` (Standard defaults both to `true`). Enable them
> explicitly with `--run-rearr true` and `--run-fd true` when running Lite.

## All CLI Options

### External catalog (`process_extcat`)

| Option | Accepts | Default | Purpose |
|:---|:---|:---|:---|
| `--extcat-input PATH` | path | `""` | Directory containing raw catalogs |
| `--extcat-output PATH` | path | `SOURCE_CAT` | Output tile directory (also sets effective `SOURCE_CAT`) |
| `--extcat-contains TEXT` | string (repeatable) | (empty) | Case-sensitive basename token (OR matching) |
| `--extcat-recursive BOOL` | bool | `true` | Recurse into subdirectories |
| `--extcat-delimiter MODE` | `auto`/`whitespace`/`comma`/`tab` | `auto` | Raw table delimiter |
| `--extcat-header MODE` | `auto`/`present`/`absent` | `auto` | Header handling |
| `--extcat-columns LIST` | comma-sep 1-based indices | `1-18` | Ordered column projection (enables explicit mode) |
| `--extcat-ra-column N` | positive int | `5` | Raw RA column (1-based) |
| `--extcat-dec-column N` | positive int | `6` | Raw Dec column (1-based) |
| `--extcat-zp-column N` | positive int | `17` | Raw photometric-redshift column (1-based) |
| `--extcat-chunk-mib N` | positive int | `64` | MPI byte-range task size (MiB) |
| `--extcat-malformed POLICY` | `fail`/`skip` | `fail` | Malformed-row policy |
| `--extcat-existing POLICY` | `fail`/`overwrite` | `fail` | Existing-tile policy |

### Initializer (`process_init`)

| Option | Accepts | Default | Purpose |
|:---|:---|:---|:---|
| `--science-root PATH` | path | `InitConfig::SCIENCE_ROOT` | Science archive root |
| `--dq-root PATH` | path | `InitConfig::DQ_ROOT` | DQ mask archive root |
| `--output-root PATH` | path | `InitConfig::OUTPUT_ROOT` | Processing output root |
| `--dataset TARGET:PREFIX` | string (repeatable) | `InitConfig::DATASETS` | Dataset target/prefix pair |
| `--target NAME` | string | - | Single dataset target (cannot combine with `--dataset`) |
| `--prefix NAME` | string | - | Single dataset prefix (cannot combine with `--dataset`) |
| `--contains TEXT` | string (repeatable) | `InitConfig::CONTAINS` | Archive basename token (OR) |
| `--existing POLICY` | `fail`/`resume`/`overwrite` | `fail` | Existing-output policy |
| `--f77-max-path N` | non-negative int | `150` (Std) / `150` (Lite) | Max path length (0 disables) |

### Downstream (`process_main` / `process_rearr` / `process_fd`)

| Option | Accepts | Default | Purpose |
|:---|:---|:---|:---|
| `--expo-list PATH` | path | `""` | Exposure-list file (or positional arg) |
| `--rearr-output-dir PATH` | path | `"baked"` | Rearrangement output directory |
| `--rearr-output-base PATH` | path | `""` | Rearrangement output base (empty=dataset root) |
| `--rearr-list-name NAME` | string | `"cat_gband_ori.list"` | Rearranged expo-list filename |
| `--rearr-list-dir PATH` | path | `""` | Rearranged expo-list directory |
| `--fd-expo-list PATH` | path | `""` | FD test expo-list override |
| `--fd-output-dir PATH` | path | `"fdout"` | FD test output directory |
| `--fd-output-base PATH` | path | `""` | FD test output base (empty=dataset root) |


### Known help-text mismatch

The current compiled default for `--rearr-list-name` is `cat_gband_ori.list` (`ProcessConfig::REARRANGED_EXPO_LIST_FILENAME`). The current `printUsage()` text still says `expo_rearranged.list`; that help label is stale. If this matters for a code cleanup, update the hard-coded help string to print or match the actual config default.

## Run Modes

### 1. External-catalog-only

Splits raw catalogs into tiles without any dataset:

```bash
mpirun -np 4 ./Fourier_Quad_Pipe \
  --run-extcat true --run-init false --run-main false --run-rearr false --run-fd false \
  --extcat-input /data/raw_catalogs \
  --extcat-output /data/catalogs/des_y6_chunks \
  --extcat-contains .csv --extcat-contains y6_gold
```

### 2. Initializer-only

Extracts chips and builds the directory tree:

```bash
mpirun -np 4 ./Fourier_Quad_Pipe \
  --run-init true --run-main false --run-rearr false --run-fd false \
  --science-root /data/archive/science \
  --dq-root /data/archive/dq \
  --output-root /data/work --dataset g2019:c4d_19
```

### 3. Main-only (existing exposure list)

Runs the numerical pipeline on pre-initialized data:

```bash
mpirun -np 4 ./Fourier_Quad_Pipe \
  --run-init false --run-main true \
  --expo-list /data/work/expo_g2019.list
```

### 4. Rearrangement-only

Consumes existing `_all.cat` files:

```bash
mpirun -np 4 ./Fourier_Quad_Pipe \
  --run-init false --run-main false --run-rearr true --run-fd false \
  --expo-list /data/work/expo_g2019.list
```

### 5. FD-test-only

Runs the FD shear test on existing `_all.cat` files:

```bash
mpirun -np 4 ./Fourier_Quad_Pipe \
  --run-init false --run-main false --run-rearr false --run-fd true \
  --expo-list /data/work/expo_g2019.list
```

### 6. Chained (init + main + rearr + fd)

After successful initialization, `process_main` and later phases receive the
normalized absolute `output_root/expo_<target>.list` path returned by
`process_init`. That generated path overrides `--expo-list`, the positional
argument, and configured defaults.

```bash
mpirun -np 4 ./Fourier_Quad_Pipe \
  --run-init true --run-main true --run-rearr true --run-fd true \
  --science-root /data/archive/science \
  --dq-root /data/archive/dq \
  --output-root /data/work --dataset g2019:c4d_19 \
  --existing resume
```

### 7. Batch (multiple datasets)

Multiple datasets run sequentially; omit `--expo-list` to let the driver derive
`output_root/expo_<target>.list` per dataset:

```bash
mpirun -np 4 ./Fourier_Quad_Pipe \
  --run-init true --run-main true \
  --science-root /data/archive/science \
  --dq-root /data/archive/dq \
  --output-root /data/work \
  --dataset g2013:c4d_13 --dataset g2014:c4d_14 --dataset g2019:c4d_19 \
  --contains v1 --contains v2
```

### 8. Full pipeline (all 5 phases)

```bash
mpirun -np 4 ./Fourier_Quad_Pipe \
  --run-extcat true --run-init true --run-main true --run-rearr true --run-fd true \
  --extcat-input /data/raw_catalogs \
  --extcat-output /data/catalogs/des_y6_cat \
  --science-root /data/archive/science \
  --dq-root /data/archive/dq \
  --output-root /data/work --dataset g2019:c4d_19
```

## Slurm Cluster

On a Slurm cluster, use the same executable arguments with the site launcher:

```bash
srun -n 40 ./Fourier_Quad_Pipe \
  --run-init true --run-main true \
  --science-root /data/archive/science \
  --dq-root /data/archive/dq \
  --output-root /data/work --dataset g2019:c4d_19
```

For containerized HPC deployment (Apptainer/Singularity + Slurm PMI2), see
[cpp-10-deployment.md](cpp-10-deployment.md).

## Common Pitfalls

1. **All phases false**: The driver rejects `--run-extcat false --run-init false
   --run-main false --run-rearr false --run-fd false`.
2. **Multiple datasets + single expo-list**: In downstream-only mode, one
   `--expo-list` cannot serve multiple datasets; omit it to derive per-dataset
   lists.
3. **Stage 9 without Stage 8**: `PROCESS_stage` with factor 23 but without 19
   is rejected (CatalogCombiner needs ExposureInfo chi2).
4. **Column projection mismatch**: When using `--extcat-columns`, the RA/Dec/ZP
   raw columns must appear in the projection list; the reader auto-maps them.
5. **extcat-output == extcat-input**: The output directory must not equal or be
   nested below the input directory.
6. **Lite-only parameters**: `ASTROMETRY_trivial=1`, `include_FLAT=1`,
   `include_Mask≠2`, `ext_cat=0`, `ext_PSF=1`, `PSF_type=2`, `PSF_Ms=1` are
   absent in `cpp_Lite`; only `cpp_Standard` implements them.

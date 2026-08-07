# C++ Pipeline: Docker & HPC Deployment

## Overview

The pipeline deploys in two environments:

1. **Docker** (`cpp_docker/`) - reproducible local build/runtime container
2. **HPC production runner** (`cpp_docker/runner/`) - reusable Slurm + Apptainer/Singularity deployment with validation helpers
3. **HPC direct mode** (CFQManual generation path) - a compact Slurm job that acquires/builds the SIF, binds only needed paths, compiles the mounted pipeline when needed, and launches it directly with `srun --mpi=pmi2`

Both HPC styles use the same OCI image/SIF execution model. Choose complexity based on the user's request rather than always reproducing the production runner.

## Docker Environment

### Directory: `cpp_docker/`

| File | Purpose |
|:---|:---|
| `Dockerfile` | Builds the runtime image |
| `compose.yaml` | Docker Compose service definition |
| `compose.optional.yaml` | Optional mounts (extcat input, rearr/fd output, expolist) |
| `pixi.toml` / `pixi.lock` | Pixi environment spec |
| `.env.example` | Environment variable template |
| `scripts/verify-image.sh` | Image verification script |
| `scripts/check-public-repo.sh` | Repo visibility check |
| `SOURCES.md` / `THIRD_PARTY_NOTICES.md` | Provenance and licenses |

### Runtime contract

| Component | Version |
|:---|:---|
| Base image | Rocky Linux 8.10 |
| G++ | 12.3.0 (conda-forge) |
| OpenMPI | 4.1.8 (Slurm PMI2 direct-launch) |
| CFITSIO | 4.6.4 |
| FFTW | 3.3.11 |
| Eigen | 3.4.0 |
| LAPACK / OpenBLAS | 3.11.0 / 0.3.33 |

### Build and verify

```bash
cd cpp_docker
cp .env.example .env
# Edit .env: set CPP_SOURCE_HOST, catalog paths, PROCESS_DATA_HOST

docker build --platform linux/amd64 --target runtime \
  --build-arg BUILD_JOBS=4 \
  -t cpppipeline-dev:gxx12.3-openmpi4.1.8-pmi2 .

bash scripts/verify-image.sh cpppipeline-dev:gxx12.3-openmpi4.1.8-pmi2
bash scripts/check-public-repo.sh
```

### Local compilation in container

```bash
cp .env.example .env
# Fill every host path
docker compose run --rm FourierQuad-CPP
make -C /workspace/src_pipe clean
make -C /workspace/src_pipe -j4
```

### Docker `.env` file

The `.env` file defines host-to-container path mappings:

| Variable | Purpose |
|:---|:---|
| `IMAGE_NAME` | Docker image tag |
| `BUILD_JOBS` | Parallel build jobs |
| `CPP_SOURCE_HOST` | Host path to `cpp_Standard/` (or `cpp_Lite/`) |
| `SCIENCE_ROOT_HOST` / `SCIENCE_ROOT_CONTAINER` | Science archive bind |
| `DQ_ROOT_HOST` / `DQ_ROOT_CONTAINER` | DQ mask archive bind |
| `ASTROMETRY_CAT_HOST` / `ASTROMETRY_CAT_CONTAINER` | Gaia catalog bind |
| `SOURCE_CAT_HOST` / `SOURCE_CAT_CONTAINER` | External source catalog bind |
| `FLAT_PATH_HOST` / `FLAT_PATH_CONTAINER` | Flat calibration bind |
| `PROCESS_DATA_HOST` / `PROCESS_DATA_CONTAINER` | Processing data (writable) |

Optional mounts (include `compose.optional.yaml` to activate):
- `EXTCAT_INPUT_HOST`, `REARR_OUTPUT_HOST`, `EXPOLIST_DIR_HOST`, `FD_OUTPUT_HOST`

### Bind interface

| Mount | Container path | Access |
|:---|:---|:---|
| Source | `/workspace/src_pipe` | read/write |
| Science archive | `SCIENCE_ROOT_CONTAINER` | read-only |
| DQ mask archive | `DQ_ROOT_CONTAINER` | read-only |
| Astrometry catalog | `ASTROMETRY_CAT_CONTAINER` | read-only |
| Source catalog | `SOURCE_CAT_CONTAINER` | read-only |
| Flat calibration | `FLAT_PATH_CONTAINER` | read-only |
| Processing data | `PROCESS_DATA_CONTAINER` | read/write |

Container catalog destinations must match the compile-time strings in
`LensingConfig.hpp` (`SOURCE_CAT`, `ASTROMETRY_CAT`, `FLAT_PATH`).


## Choosing the HPC Style

- Use the **production runner** when the user wants reusable environment files, immutable-image/checksum verification, smoke tests, or the full helper-script workflow. See [cpp-10-runner-generation.md](cpp-10-runner-generation.md).
- Use **direct HPC mode** when the user asks for a simple directly runnable Slurm script, automatic SIF acquisition/build, explicit binds, and the final pipeline command without the validation stack. See [cpp-10-direct-hpc.md](cpp-10-direct-hpc.md).

Important: the current runtime image contains the toolchain but does not copy `cpp_Standard`/`cpp_Lite` into the image. Direct mode therefore binds a live source tree and ensures `Fourier_Quad_Pipe` is built before MPI execution.

## HPC Production Runner Deployment

### Directory: `cpp_docker/runner/`

| File | Purpose |
|:---|:---|
| `cpppipeline.env.example` | HPC environment template |
| `cpppipeline.slurm` | Main pipeline Slurm script |
| `compile-pipeline.slurm` | Compilation Slurm script |
| `build-sif.slurm` | SIF build from Docker archive |
| `pull-sif.sh` | SIF pull from registry |
| `mpi-smoke-test.slurm` | MPI smoke test |
| `run-apptainer.sh` | Container launcher + `--check` validation |
| `inspect-cluster-mpi.sh` | Read-only MPI compatibility audit |
| `site-env.example.sh` | Optional site-specific module setup |

### Site prerequisites

- x86_64 Linux compute nodes
- Slurm with `pmi2` listed by `srun --mpi=list`
- Apptainer or Singularity on compute nodes
- One shared filesystem visible at identical paths on all nodes
- Routable TCP between allocated nodes

### Process boundary

```
srun --mpi=pmi2  ->  run-apptainer.sh  ->  apptainer exec --cleanenv  ->  Fourier_Quad_Pipe
```

The cluster never loads a host compiler or host OpenMPI for the application.

### Acquire the SIF

**Option A: From Docker archive**
```bash
# Set CPP_DOCKER_ARCHIVE, CPP_SIF, APPTAINER_CACHE_DIR, APPTAINER_TMP_DIR
sbatch build-sif.slurm
```

**Option B: From registry**
```bash
# Set OCI_IMAGE_URI (digest-pinned)
bash pull-sif.sh
```

Both paths:
- Build/pull to a temporary file, atomically rename the finished SIF
- Create `${CPP_SIF}.sha256` sidecar
- Refuse to overwrite an existing SIF

Copy the first field of the sidecar to `CPP_SIF_SHA256_EXPECTED` for production.

### Generate `cpppipeline.env`

Copy the example and edit every host path:

```bash
cp cpppipeline.env.example cpppipeline.env
```

Key variables to set:

| Variable | Purpose |
|:---|:---|
| `OCI_IMAGE_URI` | Digest-pinned registry image URI |
| `CPP_SIF` | Path to the SIF file |
| `CPP_SIF_SHA256_EXPECTED` | Expected SHA256 (from sidecar) |
| `CPP_SOURCE_HOST` / `CPP_SOURCE_CONTAINER` | Pipeline source bind |
| `SCIENCE_ROOT_HOST` / `SCIENCE_ROOT_CONTAINER` | Science archive (optional) |
| `DQ_ROOT_HOST` / `DQ_ROOT_CONTAINER` | DQ mask archive (optional) |
| `ASTROMETRY_CAT_HOST` / `ASTROMETRY_CAT_CONTAINER` | Gaia catalog |
| `SOURCE_CAT_HOST` / `SOURCE_CAT_CONTAINER` | External source catalog |
| `FLAT_PATH_HOST` / `FLAT_PATH_CONTAINER` | Flat calibration |
| `PROCESS_DATA_HOST` / `PROCESS_DATA_CONTAINER` | Processing data (writable) |
| `CPP_EXPO_LIST_CONTAINER` | Default exposure list path |
| `APPTAINER_CACHE_DIR` / `APPTAINER_TMP_DIR` | Apptainer cache/tmp |
| `MPI_LAUNCH_MODE` | `srun` (default) |
| `SLURM_MPI_TYPE` | `pmi2` |
| `CPP_BUILD_JOBS` | Build parallelism |
| `CPP_MAKE_CLEAN` | `1` = clean before build |
| `CPP_EXECUTABLE` | Path to `Fourier_Quad_Pipe` in container |

Optional mounts (set `*_HOST` only for mounts you need):
- `EXTCAT_INPUT_HOST`, `REARR_OUTPUT_HOST`, `EXPOLIST_DIR_HOST`, `FD_OUTPUT_HOST`

### Validation order

1. **Image and bind check:**
   ```bash
   bash run-apptainer.sh --check
   ```

2. **Compile the pipeline:**
   ```bash
   sbatch compile-pipeline.slurm
   ```

3. **One-node smoke test (2 ranks):**
   ```bash
   sbatch --nodes=1 --ntasks=2 --ntasks-per-node=2 mpi-smoke-test.slurm
   ```

4. **Multi-node smoke test:**
   ```bash
   sbatch mpi-smoke-test.slurm
   ```

5. **Production pipeline run** (after reviewing catalog paths and exposure list):
   ```bash
   sbatch cpppipeline.slurm
   ```

### Running the pipeline on HPC

Without script arguments, the executable receives
`${PROCESS_DATA_CONTAINER}/expo_list.list`. Additional arguments after the
script name are passed to `Fourier_Quad_Pipe`:

```bash
sbatch cpppipeline.slurm --run-init true --run-main true \
  --science-root /data/archive/science --dq-root /data/archive/dqmask \
  --output-root /data/DataProcess --dataset g2019:c4d_19 --contains v1
```

### Typical shared layout

```
/shared/project/cpppipeline/
├── code/           (cpp_Standard/ or cpp_Lite/)
├── runner/         (Slurm scripts + env)
├── images/         (SIF files)
├── apptainer-cache/
├── apptainer-tmp/
├── scratch/
└── data/
    ├── Science/
    ├── DQMask/
    ├── AstroDir/
    ├── ExtSrcDir/
    ├── FlatDir/
    └── DataProcess/
```

## Auto-Generating HPC Files

For the production runner, load [cpp-10-runner-generation.md](cpp-10-runner-generation.md). For a lightweight one-job Slurm deployment, load [cpp-10-direct-hpc.md](cpp-10-direct-hpc.md). Do not mix the production validation framework into direct mode unless the user asks for it.

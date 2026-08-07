# Direct HPC Slurm Deployment — Lightweight Path

Use this reference when the user wants a **simple directly runnable Slurm job** rather than the production `cpp_docker/runner` validation stack. Typical requests say things like “直接帮我生成能 sbatch 的脚本”, “不要 smoke test/大量校验”, “自动拉镜像并运行”, or “一份 Slurm 就够了”.

This path intentionally reuses the production runner's **technical contract** (container paths, binds, PMI2 launch model) without reproducing its checksum checks, image-identity tests, MPI smoke tests, helper scripts, or defensive validation framework.

## What the generated job should be able to do

A direct job may perform all of these in one script when requested:

1. initialize the site's Apptainer/Singularity environment or module;
2. acquire a SIF from an OCI registry, or build a SIF from a supplied Docker/OCI archive;
3. bind the chosen `cpp_Standard` or `cpp_Lite` source tree;
4. bind only the input/output/catalog/calibration directories required by the requested phases;
5. compile `Fourier_Quad_Pipe` inside the SIF when the mounted source does not already contain a usable executable or when the user requests a rebuild;
6. construct the exact pipeline CLI from the user's phase/data requirements;
7. launch MPI ranks with Slurm `srun --mpi=pmi2` and one Apptainer process per rank.

If the agent has shell/SSH access to the HPC and the user asked it to perform the deployment, it may create directories/files, run the SIF acquisition command, submit with `sbatch`, and inspect the submitted job. If it only has text/file-generation capability, it must create the runnable files/commands and not claim remote execution happened.

## Direct mode versus production runner

Choose **direct mode** when the user prioritizes a small operational job script and has already supplied enough site/resource/path information. Choose [`cpp-10-runner-generation.md`](cpp-10-runner-generation.md) instead when the user asks for the production runner, reusable multi-job deployment, immutable-image verification, checksum enforcement, smoke tests, or a production validation chain.

Direct mode should normally omit:

- `run-apptainer.sh --check`;
- image ID / G++ / OpenMPI version assertions;
- SIF SHA256 verification and sidecars;
- `inspect-cluster-mpi.sh`;
- one-node or multi-node smoke-test jobs;
- helper `die()`/array type-check frameworks copied from the production runner;
- fields unrelated to the requested phases.

Do **not** omit required execution plumbing merely to shorten the file: source binding, required data binds, executable build/availability, PMI2 state, and the actual pipeline CLI still matter.

## Critical image fact: the image does not contain the pipeline source

The current `cpp_docker/Dockerfile` builds the scientific toolchain and creates `/workspace/src_pipe`, but it does **not** `COPY` `cpp_Standard` or `cpp_Lite` into the runtime image. Therefore:

- acquiring the SIF alone is not enough to run `Fourier_Quad_Pipe`;
- bind the user's live source tree, normally `CPP_SOURCE_HOST:/workspace/src_pipe:rw`;
- either require an already-built `${CPP_SOURCE_HOST}/Fourier_Quad_Pipe`, or compile it inside the SIF before the MPI launch;
- a source/config change that affects compile-time behavior requires rebuilding the executable, not rebuilding the SIF.

Default direct-mode behavior when the user's intent is “make it runnable” is: **compile if the executable is missing; rebuild unconditionally only when the user requests it or the task changed compile-time source/config.**

## SIF acquisition

### Registry image → SIF

When compute/login nodes can reach the registry and the SIF does not exist:

```bash
apptainer pull "$CPP_SIF" "docker://$OCI_IMAGE_URI"
```

`apptainer pull` converts the OCI/Docker image to a SIF. Do this once before `srun`, never once per MPI rank. Prefer a shared SIF path visible to all allocated nodes.

If the cluster blocks registry access on compute nodes, do not pretend the in-job pull will work. Generate a separate login-node acquisition command or use a pre-staged archive/SIF.

### Docker archive → SIF

When a `docker save` archive is already present on the HPC/shared filesystem, a direct build can use an Apptainer-supported archive URI, for example:

```bash
apptainer build "$CPP_SIF" "docker-archive:$CPP_DOCKER_ARCHIVE"
```

Use the runtime's actual accepted transport spelling if the installed Apptainer/Singularity version differs. The important contract is **archive → one shared SIF before MPI launch**.

### Existing SIF

If the user gives an existing SIF, skip acquisition. Do not force a pull/build just because the production runner supports it.

## Minimal configuration style

For a one-off job, prefer variables near the top of the `.slurm` file. A separate env file is optional.

Use a separate small Bash env file when the user wants reusable paths across jobs. The packaged template is:

- `../templates/runner/direct-hpc.env.example`

The direct env is ordinary Bash and may contain only the variables actually needed. It does **not** need the complete production `cpppipeline.env` schema.

## Required path model

Keep host and container paths explicit. Typical stable container destinations are:

| Purpose | Typical container path | Access |
|---|---|---|
| selected source tree | `/workspace/src_pipe` | `rw` if compiling |
| science archive | `/data/archive/science` | `ro` |
| DQ mask archive | `/data/archive/dqmask` | `ro` |
| astrometry catalog | `/data/catalogs/AstroDir` | `ro` |
| source catalog | `/data/catalogs/ExtSrcDir` | normally `ro`; `rw` if `process_extcat` writes directly there |
| flat calibration | `/data/calib/FlatDir` | `ro` |
| processing/output root | `/data/DataProcess` | `rw` |
| raw external catalog input | `/data/catalogs/ExtCatInput` | `ro` |
| rearr output override | `/data/output/rearr` | `rw` |
| exposure-list override | `/data/output/expolist` | `rw` |
| FD output override | `/data/output/fdout` | `rw` |

Do not mount unused inputs solely because the production runner template contains them.

### Phase-driven bind selection

- `--run-extcat true`: bind raw extcat input and ensure the selected extcat output/SOURCE_CAT destination is writable.
- `--run-init true`: bind science and DQ roots actually passed to `--science-root` / `--dq-root`; processing output must be writable.
- `--run-main true`: bind processing data, external source catalog, astrometry catalog, and any calibration paths required by the selected compiled Standard/Lite branch.
- `--run-rearr true`: output may stay under processing data; add a dedicated rearr bind only for an external output path.
- `--run-fd true`: output may stay under the derived/default tree; add a dedicated FD bind only for an external output path.

Consult the selected variant's live config headers before assuming a compile-time catalog/calibration destination. `SOURCE_CAT` is special in the current driver because `--extcat-output` is copied into `LensingConfig::SOURCE_CAT` at runtime; when binding an arbitrary source-catalog container path for `process_main`, pass `--extcat-output "$SOURCE_CAT_CONTAINER"`. `ASTROMETRY_CAT` has no equivalent CLI override in the documented implementation, so its bind destination must match the compiled path or the config must be edited and the pipeline rebuilt.

## Compile step inside the SIF

The image already provides its compiler/MPI/scientific stack. A minimal compile command is:

```bash
apptainer exec \
  --bind "$CPP_SOURCE_HOST:$CPP_SOURCE_CONTAINER:rw" \
  "$CPP_SIF" \
  make -C "$CPP_SOURCE_CONTAINER" -j"$BUILD_JOBS"
```

Run `make clean` first only when the user requests a clean rebuild or when stale objects could invalidate the requested source/config change. Do not automatically clean on every one-off run.

The source tree and executable must be on storage visible to every rank. Compile once before `srun`, not per rank.

## MPI launch contract

The current portable image contains OpenMPI 4.1.8 built for Slurm PMI2 direct launch. The intended process boundary is:

```text
srun --mpi=pmi2
  -> apptainer exec
  -> Fourier_Quad_Pipe
```

Do not load or invoke a host OpenMPI to launch the application. Site modules may be needed to expose Slurm or Apptainer, but avoid MPI modules that inject conflicting `OMPI_*`, `PMIX_*`, or library paths.

### Clean environment and PMI forwarding

The production runner deliberately starts Apptainer with `--cleanenv` and forwards scheduler/PMI variables plus `OMPI_MCA_ess=pmi`. A direct script should retain that behavior in a compact form when using `--cleanenv`:

```bash
MPI_ENV_ARGS=()
while IFS='=' read -r name value; do
    case "$name" in
        SLURM_*|PMI_*|PMI2_*) MPI_ENV_ARGS+=(--env "$name=$value") ;;
    esac
done < <(env)
MPI_ENV_ARGS+=(--env OMPI_MCA_ess=pmi)
```

Then launch:

```bash
srun --mpi=pmi2 \
  apptainer exec --cleanenv --no-home \
  "${MPI_ENV_ARGS[@]}" \
  "${BIND_ARGS[@]}" \
  --pwd "$PROCESS_DATA_CONTAINER" \
  "$CPP_SIF" \
  "$CPP_EXECUTABLE" "${PIPELINE_ARGS[@]}"
```

This forwarding is operational MPI plumbing, not an optional “test”. If the user explicitly chooses to run without `--cleanenv`, explain that this is shorter but less isolated from host MPI/environment contamination.

## Pipeline CLI generation

Do not stop at container execution. Build `PIPELINE_ARGS` from the user's requested phases and paths using [`cpp-02-build-and-run.md`](cpp-02-build-and-run.md) and [`cpp-03a-runtime-parameters.md`](cpp-03a-runtime-parameters.md).

Prefer Bash arrays so paths and repeated options remain correctly quoted:

```bash
PIPELINE_ARGS=(
  --run-extcat false
  --run-init true
  --run-main true
  --run-rearr false
  --run-fd false
  --extcat-output "$SOURCE_CAT_CONTAINER"
  --science-root "$SCIENCE_ROOT_CONTAINER"
  --dq-root "$DQ_ROOT_CONTAINER"
  --output-root "$PROCESS_DATA_CONTAINER"
  --dataset "gband:c4d_"
  --contains v1
)
```

Do not invent a positional exposure list when `process_init` is enabled: current `main.cpp` uses the generated initializer exposure list for downstream phases. For downstream-only runs, follow the actual CLI/list rules in the build/run reference.

## Slurm resource generation

Resource directives come from the user/site policy, not from Fourier_Quad scientific parameters. Generate exactly the requested resources and preserve known site limits. A minimal direct job usually needs only:

```bash
#SBATCH --partition=...
#SBATCH --nodes=...
#SBATCH --ntasks=...
#SBATCH --ntasks-per-node=...
#SBATCH --cpus-per-task=1
#SBATCH --time=...
#SBATCH --output=...
#SBATCH --error=...
```

Do not add unrelated test allocations. Ensure `--ntasks` and `--ntasks-per-node` are consistent with the site limits supplied by the user.

## Minimal safety checks allowed in direct mode

Direct mode is not “zero checking”. Keep short checks that prevent obvious destructive or confusing failures, for example:

```bash
set -euo pipefail
mkdir -p "$(dirname "$CPP_SIF")" "$PROCESS_DATA_HOST"
command -v apptainer >/dev/null
[[ -d "$CPP_SOURCE_HOST" ]]
```

Avoid turning these into a large validation framework. The goal is runnable and readable.

## Generation workflow for the agent

1. Determine Standard vs Lite and requested phases.
2. Collect the Slurm resources and site runtime/module requirement from supplied context; do not fabricate cluster-specific values.
3. Resolve image source: existing SIF, registry OCI URI, or archive.
4. Resolve source host path and whether compilation is needed.
5. Resolve only phase-required host paths and stable container destinations.
6. Build `BIND_ARGS` and `PIPELINE_ARGS` as arrays.
7. Acquire/build SIF once before `srun` when necessary.
8. Compile once before `srun` when necessary.
9. Compactly forward Slurm/PMI environment and set `OMPI_MCA_ess=pmi`.
10. Launch one container process per Slurm rank with `srun --mpi=pmi2`.
11. If remote execution was requested and available, submit with `sbatch` and report the actual job ID/status; otherwise return the generated file and exact submit command.

## Packaged templates

- `../templates/runner/direct-pipeline.slurm.example` — single-file direct job with inline configuration.
- `../templates/runner/direct-hpc.env.example` — optional small reusable env file.

Treat these as starting points, not mandatory field lists. Delete unused bind variables and CLI options when generating the user's final script.

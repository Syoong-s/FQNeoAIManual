# Production Runner Environment Generation Contract

Use this reference when the user asks the agent to generate or adapt the **production runner** described by this manual: Docker `.env`, HPC `cpppipeline.env`, `site-env.sh`, or the validated runner Slurm files. Start from the packaged templates rather than synthesizing field names from memory.

If the user explicitly wants a small directly runnable Slurm script without smoke tests/checksum/image validation, route to [cpp-10-direct-hpc.md](cpp-10-direct-hpc.md) instead.

## Templates shipped with this skill

- `../templates/docker/.env.example`
- `../templates/runner/cpppipeline.env.example`
- `../templates/runner/site-env.example.sh`

## `cpppipeline.env` is executable Bash configuration

`cpppipeline.slurm` loads the file with `source`. Preserve Bash syntax:

- arrays must remain indexed arrays, e.g. `HPC_MODULES=()`, `SRUN_ARGS=()`, `HPC_EXTRA_BINDS=()`;
- quoted parameter expansion such as `CPP_EXECUTABLE="${CPP_SOURCE_CONTAINER%/}/Fourier_Quad_Pipe"` is valid and should not be flattened incorrectly;
- do not emit JSON/YAML syntax into this file;
- paths consumed by the Slurm wrapper as container paths must be absolute.

## Mandatory core fields

Set/verify these for an HPC runner:

| Field | Requirement |
|---|---|
| `CPP_SIF` | Existing/shared target SIF path |
| `CPP_SOURCE_HOST` | Source tree visible from compute nodes |
| `CPP_SOURCE_CONTAINER` | Absolute container path, normally `/workspace/src_pipe` |
| `PROCESS_DATA_HOST` | Writable processing area visible from compute nodes |
| `PROCESS_DATA_CONTAINER` | Absolute container path, normally `/data/DataProcess` |
| `CPP_EXPO_LIST_CONTAINER` | Absolute exposure-list path in container |
| `APPTAINER_CACHE_DIR`, `APPTAINER_TMP_DIR` | Site-appropriate writable locations |
| `MPI_LAUNCH_MODE` | Must be `srun` for the supplied Slurm wrapper |
| `SLURM_MPI_TYPE` | Must be `pmi2`; wrapper checks `srun --mpi=list` |
| `CPP_EXECUTABLE` | Absolute path to `Fourier_Quad_Pipe` in container |

`OCI_IMAGE_URI`, `CPP_DOCKER_ARCHIVE`, and `CPP_SIF_SHA256_EXPECTED` are needed according to how the SIF is acquired/verified. For production registry use, prefer a digest-pinned OCI URI.

## Phase-dependent binds

Do not require every optional bind. Infer from the requested phases and data layout:

- `--run-extcat true` → normally set `EXTCAT_INPUT_HOST/CONTAINER` plus source-catalog output bind.
- `--run-init true` → science/DQ roots must resolve either via explicit binds and CLI container paths or via an already mounted data tree.
- `--run-main true` → astrometry catalog and external source catalog paths must resolve when the compiled branch uses them; processing data must be writable.
- `--run-rearr true` → set `REARR_OUTPUT_HOST/CONTAINER` only if output is intentionally outside the default processing-data tree.
- `--run-fd true` → set `FD_OUTPUT_HOST/CONTAINER` only if output is intentionally outside the default processing-data tree.
- `EXPOLIST_DIR_HOST/CONTAINER` is optional when exposure lists already live below `PROCESS_DATA`.

## Compile-time path coupling

`ASTROMETRY_CAT`, `SOURCE_CAT`, `FLAT_PATH`, and `PSF_PATH` may be compiled into the selected Standard branch. A container bind must make the compiled path meaningful, or the caller must use a supported runtime override where one exists. In particular, external-catalog output is runtime-overridable through `RuntimeOptions`; not every calibration/scientific path is.

Before generating a runner for an edited checkout, inspect the selected variant's current config headers rather than assuming the documented defaults.

## Slurm wrapper invariants

The supplied `cpppipeline.slurm`:

1. sources `cpppipeline.env`;
2. optionally sources `SITE_ENV_SCRIPT`;
3. requires `HPC_MODULES` and `SRUN_ARGS` to be Bash indexed arrays;
4. requires `MPI_LAUNCH_MODE=srun` and `SLURM_MPI_TYPE=pmi2`;
5. verifies container-path fields are absolute;
6. checks that `srun --mpi=list` advertises `pmi2`;
7. runs `run-apptainer.sh --check` before launch;
8. launches one container process per Slurm rank through `srun`.

When adapting `#SBATCH` resources, respect the target cluster's site policy. Resource counts are site/user inputs, not pipeline scientific constants.

## Generation procedure

1. Determine variant (`cpp_Standard` or `cpp_Lite`) and source-tree host path.
2. Determine which top-level phases will run.
3. Collect only the host paths needed by those phases plus the SIF/cache/tmp locations.
4. Copy the packaged template and substitute host/site values; keep container paths stable where practical.
5. Ensure every host path is visible from **all** allocated nodes and writable where required.
6. Preserve array syntax and runner invariants.
7. Generate/adjust `cpppipeline.slurm` resource directives separately from `cpppipeline.env`; do not hide CPU/node requests in the env file.
8. Validate in order: runner `--check` → compile → one-node MPI smoke test → multi-node smoke test → representative pipeline job.

## Output checklist for an AI-generated runner

Return or create:

- exact generated file path(s);
- selected variant and phases;
- unresolved placeholders, if any;
- host→container bind table;
- Slurm resource request and MPI mode;
- validation commands;
- whether any compile-time path/config change requires rebuilding before launch.

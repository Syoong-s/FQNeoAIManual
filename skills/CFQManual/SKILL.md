---
name: CFQManual
description: Background manual for Fourier Quad Pipeline Neo (C++) (cpp_Standard, cpp_Lite, cpp_docker). Use for architecture, exact parameter selection/tuning, CLI/run setup, production runner generation, lightweight direct HPC Slurm deployment, debugging, and developer code changes. Route to only the smallest relevant reference set; verify live source symbols before edits. Use when the user asks to understand, configure, run, tune, debug, or modify a compatible Neo (C++) Fourier Quad pipeline.
---

# CFQManual — Fourier Quad Pipeline Neo (C++)

Use this skill to let an agent with no prior project knowledge understand, configure, run, deploy, debug, or modify Fourier Quad Pipeline Neo (C++) without loading the whole manual into context.

## Core operating rule

**Route first, read second. Never load every reference by default.** Start from the user's concrete task, select the minimum reference set below, then inspect the live checkout when code is available. Detailed references intentionally retain implementation-level information; `SKILL.md` is only the index and operating policy.

## Pipeline identity

The unified executable is `Fourier_Quad_Pipe`. The driver coordinates five top-level phases in fixed order:

`process_extcat → process_init → process_main → process_rearr → process_fd`

`process_main` contains nine prime-gated numerical stages. `cpp_Standard` contains the full branch set; `cpp_Lite` physically removes eight frozen branches, so Lite is not merely Standard with different constants.

## Task router

Load only the rows needed for the task.

| Task | Minimum references |
|---|---|
| Architecture, project orientation, data flow, MPI model | `cpp-01-overview.md` |
| Build, CLI, phase combinations, local/MPI execution | `cpp-02-build-and-run.md` |
| User names a parameter or asks where it lives | `cpp-03-parameters.md` → one detailed parameter file |
| User gives an outcome and asks which value(s) to tune | `cpp-03d-tuning-guide.md` + one detailed parameter file |
| External catalog repartition/projection | `cpp-04-process-extcat.md` + `cpp-03a-runtime-parameters.md` |
| Archive initialization / exposure lists | `cpp-05-process-init.md` + `cpp-03a-runtime-parameters.md` |
| `process_main` orchestration or stage dependency | `cpp-06-process-main.md` |
| One numerical stage | `cpp-06-process-main.md` + exactly one `cpp-06-stage-*.md` |
| Catalog rearrangement | `cpp-07-process-rearr.md` + `cpp-03c-rearr-fd-parameters.md` |
| FD shear test | `cpp-08-process-fd.md` + `cpp-03c-rearr-fd-parameters.md` |
| Standard vs Lite choice/migration | `cpp-09-standard-vs-lite.md` |
| Docker/Apptainer/HPC overview or choose deployment style | `cpp-10-deployment.md` |
| Generate production Docker `.env` / HPC `cpppipeline.env` using production runner | `cpp-10-runner-generation.md` + packaged template |
| Generate a lightweight directly runnable HPC Slurm job (SIF acquire/build + binds + compile-if-needed + `srun`) | `cpp-10-direct-hpc.md` + direct-HPC template(s) |
| Modify implementation / add feature / debug source | `cpp-11-code-modification.md` + only the affected phase/stage reference |

If scope is genuinely unclear, load `cpp-01-overview.md` to orient once. Do **not** make overview mandatory for already-localized tasks.

## Parameter router

`cpp-03-parameters.md` is itself a small index:

- `cpp-03a-runtime-parameters.md` — phase switches, CLI/runtime paths, extcat parsing, initializer defaults.
- `cpp-03b-main-parameters.md` — `LensingConfig` scientific/numerical constants and implementation-local knobs.
- `cpp-03c-rearr-fd-parameters.md` — rearrangement layout plus FD statistics/cuts/binning.
- `cpp-03d-tuning-guide.md` — goal → likely knob, coupling constraints, and value-selection workflow.

### Runtime versus rebuild

Do not describe the config headers as one global precedence chain. `main.cpp` seeds `ProcessConfig::RuntimeOptions` from compiled defaults, and CLI options override only fields represented in `RuntimeOptions`. Most `LensingConfig`/`FDConfig` scientific values remain compile-time and require rebuilding. Some tunable values are implementation-local; for example current Stage 7 defines `PSFr_ratio=0.75f` inside `ShearMeasurement::getShear` rather than in `LensingConfig`.

## Numerical-stage router

`cpp-06-process-main.md` contains orchestration/support information. Load only one stage file for a localized task:

1. `cpp-06-stage-01-preprocess.md`
2. `cpp-06-stage-02-astrometry.md`
3. `cpp-06-stage-03-source.md`
4. `cpp-06-stage-04-fft-stars.md`
5. `cpp-06-stage-05-psf.md`
6. `cpp-06-stage-06-fft-galaxies.md`
7. `cpp-06-stage-07-shear.md`
8. `cpp-06-stage-08-exposure.md`
9. `cpp-06-stage-09-combine.md`

`PROCESS_stage` uses primes `2,3,5,7,11,13,17,19,23`; a stage runs when the product is divisible by its prime. Stage 9 requires Stage 8 and the current source rejects factor 23 without factor 19.

## Code inspection policy

When pipeline source code is available for a modification or debugging task:

1. Identify `cpp_Standard` or `cpp_Lite` before proposing a patch.
2. Search for the exact symbol/function/error text in that variant.
3. Read declaration, implementation, direct call sites, and output/schema consumers as needed.
4. For Lite, check `cpp-09-standard-vs-lite.md` before touching any branch that may have been deleted.

Do not invent a config symbol merely because a concept sounds configurable.

## Parameter-adjustment response contract

When asked to tune values, give the agent/user enough information to act precisely:

- exact variant, file, namespace/symbol;
- current value (from live source when available);
- proposed value or range and why it affects the requested outcome;
- coupled parameters/invariants that must move with it;
- runtime vs rebuild classification;
- smallest validation that can falsify the change.

If the desired value cannot be inferred safely from the request, prefer a bounded experiment/scan over a guessed single number.

## HPC deployment mode contract

Do not assume every HPC request wants the production runner pattern. First classify the request:

- **Production/reusable/validated runner** — load `cpp-10-runner-generation.md`; generate `cpppipeline.env` and adapt the production runner. Preserve Bash arrays and its `srun + pmi2` invariants.
- **Direct/lightweight/one-job Slurm** — load `cpp-10-direct-hpc.md`; generate only the minimum operational plumbing needed to acquire/build the SIF, mount inputs/outputs/source, compile the mounted pipeline when needed, forward Slurm/PMI state into a clean container, and launch `Fourier_Quad_Pipe`. Do not copy smoke tests, checksum/image-version checks, or the production runner validation framework unless the user asks for them.

For either mode, keep host paths separate from container paths and report unresolved site-specific values rather than fabricating them. The current portable image is a toolchain/runtime image: it creates `/workspace/src_pipe` but does not copy the C++ pipeline source into the image, so direct HPC execution must bind a source tree and ensure `Fourier_Quad_Pipe` has been built there.

## Developer-change contract

For code iteration, load `cpp-11-code-modification.md` plus the exact affected phase/stage reference, then inspect the checkout. Prefer the smallest coherent patch. Do not mirror a Standard change into Lite until checking whether that code path exists there. After a source/compile-time change, rebuild the affected variant and validate the branch exercised by the change. Current variant Makefiles expose `all`, `clean`, and the focused Stage 7 `test-point-source-statistics` target; do not assume historical `test-extcat-reader` or `test-rearr` Make targets exist.

# Runtime / Workflow Parameters

These are the settings most likely to be changed per run. CLI values override the compiled defaults only where `main.cpp` maps an option into `ProcessConfig::RuntimeOptions`.

## ProcessConfig.hpp — Workflow Defaults

Phase switches and I/O path defaults. These seed the `RuntimeOptions` struct;
CLI options override them at runtime.

| Parameter | CLI | Default (Std) | Description |
|:---|:---|:---:|:---|
| `RUN_PROCESS_EXTCAT` | `--run-extcat` | `false` | Phase switch for external-catalog repartitioning |
| `RUN_PROCESS_INIT` | `--run-init` | `true` | Phase switch for archive initialization |
| `RUN_PROCESS_MAIN` | `--run-main` | `true` | Phase switch for the 9-stage numerical pipeline |
| `RUN_PROCESS_REARR` | `--run-rearr` | `true` | Phase switch for `_all.cat` rearrangement |
| `RUN_PROCESS_FD` | `--run-fd` | `true` | Phase switch for FD shear test |
| `EXPO_LIST` | `--expo-list` | `""` | Single exposure-list file path |
| `REARR_OUTPUT_DIRECTORY` | `--rearr-output-dir` | `"baked"` | Rearrangement output directory |
| `REARR_OUTPUT_BASE_DIRECTORY` | `--rearr-output-base` | `""` | Rearrangement output base (empty=dataset root) |
| `REARRANGED_EXPO_LIST_FILENAME` | `--rearr-list-name` | `"cat_gband_ori.list"` | Rearranged expo-list filename |

**Current source note:** the effective compiled default is `cat_gband_ori.list`. `main.cpp::printUsage()` currently contains a stale help string that says `expo_rearranged.list`; use `ProcessConfig::REARRANGED_EXPO_LIST_FILENAME` and the actual `RuntimeOptions` value when determining the effective setting.
| `REARRANGED_EXPO_LIST_DIRECTORY` | `--rearr-list-dir` | `""` | Rearranged expo-list directory |
| `FD_EXPO_LIST` | `--fd-expo-list` | `""` | FD test expo-list override |
| `FD_OUTPUT_DIRECTORY` | `--fd-output-dir` | `"fdout"` | FD test output directory |
| `FD_OUTPUT_BASE_DIRECTORY` | `--fd-output-base` | `""` | FD test output base (empty=dataset root) |

---

## ExtCatConfig.hpp — External Catalog Tiling

| Parameter | CLI | Default | Description |
|:---|:---|:---:|:---|
| `EXTCAT_INPUT_DIRECTORY` | `--extcat-input` | `""` | Root directory containing raw catalog files |
| `EXTCAT_OUTPUT_DIRECTORY` | `--extcat-output` | `= SOURCE_CAT` | Output tile directory (also updates effective `SOURCE_CAT`) |
| `EXTCAT_FILENAME_TOKENS` | `--extcat-contains` | `{}` | Case-sensitive basename filters (repeatable, OR) |
| `EXTCAT_RECURSIVE` | `--extcat-recursive` | `true` | Recurse into subdirectories |
| `EXTCAT_DELIMITER` | `--extcat-delimiter` | `auto` | `auto`/`whitespace`/`comma`/`tab` |
| `EXTCAT_HEADER_MODE` | `--extcat-header` | `auto` | `auto`/`present`/`absent` |
| `EXTCAT_MALFORMED_POLICY` | `--extcat-malformed` | `fail` | `fail`/`skip` |
| `EXTCAT_EXISTING_POLICY` | `--extcat-existing` | `fail` | `fail`/`overwrite` |
| `EXTCAT_CHUNK_MIB` | `--extcat-chunk-mib` | `64` | MPI byte-range task size (MiB) |
| `EXTCAT_TOTAL_COLUMNS` | - | `18` | Canonical DES Y6 GOLD column count (compile-time) |
| `EXTCAT_USE_EXPLICIT_COLUMNS` | `--extcat-columns` | `false` | `false`=pass-through, `true`=explicit projection |
| `EXTCAT_INPUT_COLUMNS_ONE_BASED` | `--extcat-columns` | `{1..18}` | Ordered output column projection (1-based) |
| `EXTCAT_USE_EXPLICIT_COORDINATE_COLUMNS` | - | `false` | Use explicit coordinate columns (compile-time) |
| `EXTCAT_RA_COLUMN_ONE_BASED` | `--extcat-ra-column` | `5` | Raw RA field position (1-based) |
| `EXTCAT_DEC_COLUMN_ONE_BASED` | `--extcat-dec-column` | `6` | Raw Dec field position (1-based) |
| `EXTCAT_ZP_COLUMN_ONE_BASED` | `--extcat-zp-column` | `17` | Raw photometric-redshift (`dnf_z`) position (1-based) |

**Note:** When `EXTCAT_USE_EXPLICIT_COLUMNS=true`, `process_main` auto-indexes
RA/Dec/ZP from the projection list instead of using the column constants
directly. The three field columns must appear in the projection list.

---

## InitConfig.hpp — Initializer Defaults

| Parameter | CLI | Default (Std) | Description |
|:---|:---|:---:|:---|
| `SCIENCE_ROOT` | `--science-root` | `/lustre/.../DES/g` | Science archive root |
| `DQ_ROOT` | `--dq-root` | `/lustre/.../DES/mask_v1/g_mask` | DQ mask archive root |
| `OUTPUT_ROOT` | `--output-root` | `/lustre/.../DES/g_band_v1` | Processing output root |
| `DATASETS` | `--dataset` | `{{"gband","c4d_"}}` | Dataset target/prefix pairs (repeatable) |
| `CONTAINS` | `--contains` | `{"v1"}` | Archive basename tokens (repeatable, OR) |
| `EXISTING` | `--existing` | `fail` | `fail`/`resume`/`overwrite` |
| `F77_MAX_PATH` | `--f77-max-path` | `150` (Std) / `150` (Lite) | Max path length (0 disables) |

---

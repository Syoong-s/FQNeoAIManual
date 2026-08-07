# process_init: Archive Initializer

## Purpose

`process_init` extracts Science and DQ mask chip images from compressed
`.fits.fz` archives, builds the pipeline directory tree, and publishes
top-level exposure lists. It runs **per dataset** (once for each `TARGET:PREFIX`
pair) and is the prerequisite for `process_main`.

## When to Run

- When processing raw archives for the first time
- When the dataset or archive set changes
- Before any `process_main` run that doesn't already have initialized data
- Use `--existing resume` to continue from a partially initialized dataset

## Source Files

| File | Role |
|:---|:---|
| `src/process_init/process_init.cpp` | Initializer wrapper (translates RuntimeOptions) |
| `src/process_init/Initializer.cpp` | Core initialization logic |
| `src/process_init/FitsExtractor.cpp` | `.fits.fz` archive extraction |
| `include/process_init/{process_init,Initializer,FitsExtractor}.hpp` | Headers |
| `config/InitConfig.hpp` | Default values |

## Processing Flow

For each `--output-root OUTPUT --dataset TARGET:PREFIX`:

1. **Validate existing output** (`EXISTING` policy):
   - `fail` (default): reject if output exists
   - `resume`: keep existing files, continue
   - `overwrite`: replace all existing output
2. **Scan Science archives**: Find `.fits.fz` files matching `CONTAINS` tokens
   in `SCIENCE_ROOT`. Each archive basename must contain at least one token.
3. **Extract Science chips**: Read each archive in place (never copied/removed),
   extract Science chip images by HDU, write to
   `OUTPUT/TARGET/science/<EXPOSURE>/<EXPOSURE>_<N>.fits`.
   - `<N>` is the sequential two-dimensional HDU occurrence index (1, 2, 3, ...).
4. **Extract DQ mask chips**: From `DQ_ROOT`, write to
   `OUTPUT/TARGET/dqmask/<EXPOSURE>/<EXPOSURE>_<CCDNUM>.fits`.
   - `<CCDNUM>` is the FITS `CCDNUM` header value.
5. **Write per-exposure chip lists**: Each rank writes
   `stamps/<EXPOSURE>.list` (chip image paths it produced) directly from
   extraction results - no post-hoc disk re-scan.
6. **Publish top-level lists** (rank 0 scans `stamps/`, sorts, atomically publishes):
   - `OUTPUT/expo_TARGET.list` - top-level list; each line is
     `"<OUTPUT/TARGET/stamps/<EXPOSURE>.list>" <chip count>`
   - `OUTPUT/fits_TARGET.list` - flat list of every Science chip path
   - `OUTPUT/init_TARGET_manifest.json` - initialization manifest (schema v2)

## Output Layout

```
OUTPUT/TARGET/
├── science/
│   └── <EXPOSURE>/
│       ├── <EXPOSURE>_1.fits    (Science chip, HDU occurrence index)
│       ├── <EXPOSURE>_2.fits
│       └── ...
├── dqmask/
│   └── <EXPOSURE>/
│       ├── <EXPOSURE>_<CCDNUM>.fits    (DQ mask, by CCDNUM header)
│       └── ...
└── stamps/
    ├── <EXPOSURE>.list    (per-exposure chip list)
    └── ...                (process_main intermediate products go here later)
```

Top-level files:
```
OUTPUT/
├── expo_TARGET.list               (top-level exposure list)
├── fits_TARGET.list               (flat chip image list)
└── init_TARGET_manifest.json      (manifest, schema v2)
```

## Manifest Schema (v2)

Records all active basename filters in the `filename_tokens` array. Science
chips are numbered by two-dimensional HDU occurrence; DQ chips are numbered by
`CCDNUM`. Downstream stages derive the dataset root from a Science chip path
via `getDir(image, 3)` (three levels up: `science/<EXPOSURE>/<file>` ->
`science` -> `OUTPUT/TARGET`). Per-chip DQ masks are read from
`dqmask/<EXPOSURE>/<EXPOSURE>_<CCDNUM>.fits`.

## Configuration

All parameters in `InitConfig.hpp`, overridable via CLI. See
[cpp-03a-runtime-parameters.md](cpp-03a-runtime-parameters.md).

| Parameter | CLI | Default (Std) | Purpose |
|:---|:---|:---:|:---|
| `SCIENCE_ROOT` | `--science-root` | `/lustre/.../DES/g` | Science archive root |
| `DQ_ROOT` | `--dq-root` | `/lustre/.../g_mask` | DQ mask archive root |
| `OUTPUT_ROOT` | `--output-root` | `/lustre/.../g_band_v1` | Processing output root |
| `DATASETS` | `--dataset` | `{{"gband","c4d_"}}` | Target/prefix pairs (repeatable) |
| `CONTAINS` | `--contains` | `{"v1"}` | Archive basename tokens (repeatable, OR) |
| `EXISTING` | `--existing` | `fail` | `fail`/`resume`/`overwrite` |
| `F77_MAX_PATH` | `--f77-max-path` | `150` | Max path length (0=disable) |

## Dataset Specification

Datasets are `TARGET:PREFIX` pairs:

```bash
--dataset g2013:c4d_13 --dataset g2014:c4d_14 --dataset g2019:c4d_19
```

- `TARGET` is the dataset name (used in directory and list names)
- `PREFIX` is the archive filename prefix (e.g., `c4d_` for DECam)
- Multiple datasets run sequentially; the pipeline stops at the first failure

`--target` and `--prefix` are single-dataset alternatives that cannot be
combined with `--dataset`.

## Chained Execution

When `--run-init true` is combined with downstream phases (`--run-main`,
`--run-rearr`, `--run-fd`), the initializer-generated absolute exposure list
path (`output_root/expo_<target>.list`) takes precedence over `--expo-list`,
the positional argument, and configured defaults. Each dataset gets its own
generated list.

## Failure Behavior

- Initializer failure is **collective**: if any rank fails, all ranks stop.
- The numerical phase is never entered after initialization fails.
- Use `--existing resume` to continue from a partial initialization.

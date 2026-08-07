# process_extcat: External Catalog Repartitioning

## Purpose

`process_extcat` is the first pipeline phase. It splits raw external source
catalogs (e.g., DES Y6 GOLD) into deterministic 0.1-degree sky tiles that
`process_main` reads via `SOURCE_CAT`. It runs **once** before the dataset loop
and does not require a configured dataset.

## When to Run

- When raw catalogs need to be tiled for the first time
- When the external catalog schema or projection changes
- Standalone (no dataset needed): `--run-extcat true --run-init false --run-main false`

## Source Files

| File | Role |
|:---|:---|
| `src/process_extcat/process_extcat.cpp` | Tiler implementation |
| `include/process_extcat/process_extcat.hpp` | Config struct, enums, API |
| `config/ExtCatConfig.hpp` | Default values |

## Processing Flow

1. **Discover inputs** (rank 0): Scan `EXTCAT_INPUT_DIRECTORY` for files matching
   `EXTCAT_FILENAME_TOKENS` (OR matching). Recursive if `EXTCAT_RECURSIVE`.
2. **Validate config**: Canonicalize paths, reject unsafe overlap (output
   must not equal or be nested below input), validate column mappings and
   task size.
3. **Process newline-aligned byte ranges**: Each MPI rank processes a
   `EXTCAT_CHUNK_MIB`-sized byte range of each input file. Rows are split on
   newline boundaries so no row is corrupted across ranks.
4. **Optional column projection**: If `EXTCAT_USE_EXPLICIT_COLUMNS=true`, emit
   only the columns listed in `EXTCAT_INPUT_COLUMNS_ONE_BASED` in that order.
   Otherwise, pass through all raw fields in place.
5. **Publish deterministic tiles**: Write one-degree sky tiles to
   `EXTCAT_OUTPUT_DIRECTORY`. Tile filenames match
   `gen_src_cat/query_y6gold_sync_mp_v2.py` output.

## Configuration

All parameters are in `ExtCatConfig.hpp` and overridable via CLI. See
[cpp-03a-runtime-parameters.md](cpp-03a-runtime-parameters.md) for the full runtime/extcat table.

### Key parameters

| Parameter | CLI | Default | Purpose |
|:---|:---|:---:|:---|
| `EXTCAT_INPUT_DIRECTORY` | `--extcat-input` | `""` | Raw catalog root |
| `EXTCAT_OUTPUT_DIRECTORY` | `--extcat-output` | `= SOURCE_CAT` | Tile output (also sets `SOURCE_CAT`) |
| `EXTCAT_FILENAME_TOKENS` | `--extcat-contains` | `{}` | Basename filters (repeatable, OR) |
| `EXTCAT_DELIMITER` | `--extcat-delimiter` | `auto` | `auto`/`whitespace`/`comma`/`tab` |
| `EXTCAT_HEADER_MODE` | `--extcat-header` | `auto` | `auto`/`present`/`absent` |
| `EXTCAT_USE_EXPLICIT_COLUMNS` | `--extcat-columns` | `false` | Enable column projection |
| `EXTCAT_INPUT_COLUMNS_ONE_BASED` | `--extcat-columns` | `{1..18}` | Projection order |
| `EXTCAT_RA_COLUMN_ONE_BASED` | `--extcat-ra-column` | `5` | Raw RA position |
| `EXTCAT_DEC_COLUMN_ONE_BASED` | `--extcat-dec-column` | `6` | Raw Dec position |
| `EXTCAT_ZP_COLUMN_ONE_BASED` | `--extcat-zp-column` | `17` | Raw redshift position |
| `EXTCAT_CHUNK_MIB` | `--extcat-chunk-mib` | `64` | MPI byte-range task size |
| `EXTCAT_MALFORMED_POLICY` | `--extcat-malformed` | `fail` | `fail`/`skip` |
| `EXTCAT_EXISTING_POLICY` | `--extcat-existing` | `fail` | `fail`/`overwrite` |

## Column Projection

Two modes:

### Pass-through (default)

`EXTCAT_USE_EXPLICIT_COLUMNS=false`. All raw fields are preserved in place. The
output schema width is `EXTCAT_TOTAL_COLUMNS` (18 for DES Y6 GOLD).
`process_main` reads RA/Dec/ZP at their configured positions directly.

### Explicit projection

`EXTCAT_USE_EXPLICIT_COLUMNS=true` (set by passing `--extcat-columns`).
Output contains exactly the columns listed in `EXTCAT_INPUT_COLUMNS_ONE_BASED`,
in that order. The reader auto-maps each raw index to its position in the
projection list. The required fields (RA, Dec, ZP) must appear in the
projection.

**Example:** `--extcat-columns 5,3,4,1` writes raw columns 5, 3, 4, 1 as
output columns 1-4. If RA is raw column 5, it becomes output column 1.

## Delimiter and Header Detection

- **`auto` delimiter**: chooses comma when present, otherwise tab when present,
  otherwise whitespace.
- **`auto` header**: recognizes case-insensitive `ra`/`dec` column names and
  can classify the leading record as a header. Leading blank and `#` comment
  lines are always skipped.
- **`present`**: requires a header row.
- **`absent`**: treats all records as data.

## Output

Tiles are written to `EXTCAT_OUTPUT_DIRECTORY` (which defaults to
`LensingConfig::SOURCE_CAT`, so `process_extcat` writes where `process_main`
reads). Each tile covers a 0.1° × 0.1° sky region. Filenames match the Python
`gen_src_cat` tool output.

If `process_extcat` fails, no later phase starts (the failure is collective).

## External Catalog Generation

Raw catalogs can also be generated from the DES Y6 GOLD database using the
`gen_src_cat/` Python tool:

```bash
cd gen_src_cat
python3 -m venv .venv
source .venv/bin/activate
python -m pip install numpy pyvo
python query_y6gold_sync_mp_v2.py
```

Review sky bounds, row limit, output directory, and query concurrency before
running. Set the primary `SOURCE_CAT` path in `LensingConfig.hpp`, or pass
`--extcat-output`; `EXTCAT_OUTPUT_DIRECTORY` follows that value.

# process_rearr: Catalog Rearrangement

## Purpose

`process_rearr` reads per-exposure `_all.cat` shear catalogs produced by
`process_main`, partitions them by sky region using a weighted k-d tree, and
writes spatially sorted subcatalogs. It runs **per dataset** after
`process_main` (when both are enabled), or independently on existing results.

## Source Files

| File | Role |
|:---|:---|
| `src/process_rearr/process_rearr.cpp` | Orchestration + MPI redistribution |
| `src/process_rearr/CatalogRearranger.cpp` | K-d partitioning + sorted output |
| `include/process_rearr/process_rearr.hpp` | API |
| `include/process_rearr/CatalogRearranger.hpp` | Partitioner API |
| `config/ProcessRearrConfig.hpp` | All parameters |

## Processing Flow

1. **Read catalogs** across MPI ranks: Each rank reads a subset of the
   per-exposure `_all.cat` files referenced by the exposure list.
2. **Validate schema**: Check that each row has the exact expected width
   (`allCatalogColumns(options)` = effective external width + 1 + ichi2).
3. **Build global weighted k-d partition**: Partition the full sky into
   subcatalogs targeting `TARGET_SUBCAT_ROWS` rows each, using 0.1-degree
   tiles as indivisible units. The k-d split is weighted by source count.
4. **MPI redistribution**: Redistribute complete rows so each rank owns its
   assigned partitions.
5. **Sort and write**: Each partition is sorted by Dec then RA (source
   exposure/row as deterministic tie breakers), and written as
   `subcat_NNNNNN.cat`.
6. **Summary**: Write `catalog_summary.txt` with partition count and raw
   RA/Dec bounds.

## `_all.cat` Row Format

Each row has `allCatalogColumns(options)` fields:

```
[external catalog fields] [CCD_NUM] [29 process-main fields]
         18 (default)          1       28 Stage 7 + 1 exposure chi2
```

Default total: `18 + 1 + 29 = 48` columns.

### Derived column layout (compile-time, validated with `static_assert`)

| Constant | Value | Description |
|:---|:---:|:---|
| `ichi2` | `29` | Count of 28 Stage 7 fields + exposure chi2 after CCD_NUM |
| `CCD_COLUMN_COUNT` | `1` | Fixed CCD_NUM field count |
| `ALL_CAT_TOTAL_COLUMNS` | `48` | `EXTCAT_TOTAL_COLUMNS + 1 + ichi2` |
| `externalCatalogColumns(options)` | `18` | Runtime-effective external width |
| `allCatalogColumns(options)` | `48` | Runtime-effective row width in pass-through mode |

### Column resolution

- **Pass-through mode** (`EXTCAT_USE_EXPLICIT_COLUMNS=false`): external width =
  `EXTCAT_TOTAL_COLUMNS` (18). RA/Dec read at configured positions.
- **Explicit projection mode** (`EXTCAT_USE_EXPLICIT_COLUMNS=true`): external
  width = projection list length. RA/Dec resolved through the projection.

## Configuration

Partition/layout constants are in `ProcessRearrConfig.hpp`; output paths are runtime fields from `ProcessConfig::RuntimeOptions`. See
[cpp-03c-rearr-fd-parameters.md](cpp-03c-rearr-fd-parameters.md).

### Spatial partitioning

| Parameter | Default | Description |
|:---|:---:|:---|
| `SKY_GRID_DEGREES` | `0.1` | Sky bin width in RA and Dec (°) |
| `RA_BIN_COUNT` | `3600` | Full-sky RA grid dimension |
| `DEC_BIN_COUNT` | `1800` | Full-sky Dec grid dimension |
| `SKY_TILE_COUNT` | `6480000` | Total sky tiles |
| `TARGET_SUBCAT_ROWS` | `500000` | Target rows per k-d partition |

### Output

| Parameter | Default | Description |
|:---|:---:|:---|
| `SUBCAT_PREFIX` | `"subcat_"` | Partition filename prefix |
| `SUBCAT_EXTENSION` | `".cat"` | Partition filename extension |
| `SUBCAT_ID_WIDTH` | `6` | Min zero-padded partition-ID width |
| `SUMMARY_FILENAME` | `"catalog_summary.txt"` | Summary report filename |
| `OUTPUT_PRECISION` | `10` | Significant digits for catalog values |
| `SUMMARY_PRECISION` | `4` | Fixed decimals for summary bounds |

### Error handling

| Parameter | Default | Description |
|:---|:---:|:---|
| `SKIP_MISSING_CATALOGS` | `true` | Skip absent `_all.cat` files (count + report) |
| `SKIP_MALFORMED_ROWS` | `true` | Skip malformed rows (count + report) |

When `SKIP_MISSING_CATALOGS=false` or `SKIP_MALFORMED_ROWS=false`, the pipeline
fails collectively on the first missing catalog or malformed row.

## Output

```
<resolved rearr output directory>/
├── subcat_000001.cat
├── subcat_000002.cat
├── ...
├── subcat_NNNNNN.cat
└── catalog_summary.txt
```

- Each `subcat_NNNNNN.cat` contains rows sorted by Dec then RA
- Source exposure/row are used only as deterministic tie breakers
- The shared input header is copied
- Existing same-name files are truncated (like the legacy pipeline)
- Output path: if `--rearr-output-base` is absolute, use it; otherwise it's a
  child of the dataset root. If `--rearr-output-dir` is relative, it's joined
  to the base; if empty, defaults to the base.

## Runtime Path Resolution

| CLI option | Config default | Purpose |
|:---|:---|:---|
| `--rearr-output-dir` | `"baked"` | Output directory name |
| `--rearr-output-base` | `""` | Base directory (empty = dataset root) |
| `--rearr-list-name` | `"cat_gband_ori.list"` | Rearranged expo-list filename |
| `--rearr-list-dir` | `""` | Rearranged expo-list directory |

The rearranged exposure list (`cat_gband_ori.list` by default) is consumed by
`process_fd` if `--fd-expo-list` is not set.

# process_main Detailed Stage Reference

## Stage 9: Catalog Combination

| | |
|:---|:---|
| **Prime** | 23 (requires Stage 8, prime 19) |
| **Function** | `CatalogCombiner::procComb(int iexpo)` |
| **Files** | `src/process_main/CatalogCombiner.cpp`, `include/process_main/CatalogCombiner.hpp` |
| **Config** | `LensingConfig.hpp`: `chi2_thresh`, `g1_c`, `g2_c`, `ext_cat`, column indices |

### What it does

For each exposure:
1. Read all per-chip shear catalogs (24 fields + exposure chi2)
2. Apply quality cuts:
   - `chi2_thresh`: reject exposures with high PSF chi2
   - Column-based cuts (SNR, flag, size, etc.)
3. Apply the active-branch shear correction. **Current default `ext_cat=1` sets local `g1c=g2c=0`, so the correction equations are an identity; the nonzero `LensingConfig::g1_c/g2_c` expression is commented out in this branch.** The `ext_cat=0` branch uses `gf + g_c`.
4. Merge all chips into one per-exposure catalog
5. Write `result/<EXPO>_all.cat`

### `_all.cat` row format

Each row has `EXTCAT_TOTAL_COLUMNS + 1 + ichi2` fields:
- External catalog fields (18 in pass-through mode)
- 1 CCD_NUM field
- 25 shear fields (24 source columns + exposure chi2)

Default total width: `18 + 1 + 25 = 44` columns.

### Key sub-functions

| Function | Purpose |
|:---|:---|
| `procComb` | Stage 9 main entry |
| `combineExpoCatalog` | Merge per-chip catalogs into per-exposure `_all.cat` |

---

## Calibration branch warning

Do not change `g1_c`/`g2_c` expecting the default external-catalog path to respond: with current `ext_cat=1`, `CatalogCombiner::combineExpoCatalog` explicitly initializes the local correction to zero. Inspect and modify the active branch if the intended behavior is to apply those constants.

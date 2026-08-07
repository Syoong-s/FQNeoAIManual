# Rearrangement & FD Parameters

## ProcessRearrConfig.hpp — Rearrangement Parameters

The layout/partition constants below are compile-time. Output base/directory names are **runtime** fields in `ProcessConfig::RuntimeOptions` (`--rearr-output-base`, `--rearr-output-dir`), not `ProcessRearrConfig` constants. The derived column layout is validated with `static_assert`.

### Derived Derived `_all.cat` column layout

| Parameter | Default | Description |
|:---|:---:|:---|
| `ichi2` | `25` | Count of 24 shear fields + exposure chi2 after CCD_NUM |
| `CCD_COLUMN_COUNT` | `1` | Fixed CCD_NUM field count |
| `ALL_CAT_TOTAL_COLUMNS` | `44` | `EXTCAT_TOTAL_COLUMNS + 1 + ichi2` = `18 + 1 + 25` |
| `externalCatalogColumns(options)` | `18` | Runtime-effective external width |
| `allCatalogColumns(options)` | `44` | Runtime-effective row width |

### Spatial Spatial partitioning and output

| Parameter | Default | Description |
|:---|:---:|:---|
| `SKY_GRID_DEGREES` | `0.1` | Full-sky bin width in RA and Dec (°) |
| `RA_BIN_COUNT` | `3600` | Full-sky RA grid dimension |
| `DEC_BIN_COUNT` | `1800` | Full-sky Dec grid dimension |
| `SKY_TILE_COUNT` | `6480000` | `RA_BIN_COUNT × DEC_BIN_COUNT` |
| `TARGET_SUBCAT_ROWS` | `500000` | Target rows per weighted k-d partition |
| `SUBCAT_PREFIX` | `"subcat_"` | Partition filename prefix |
| `SUBCAT_EXTENSION` | `".cat"` | Partition filename extension |
| `SUBCAT_ID_WIDTH` | `6` | Min zero-padded partition-ID width |
| `SUMMARY_FILENAME` | `"catalog_summary.txt"` | Summary report filename |
| `OUTPUT_PRECISION` | `10` | Significant digits for catalog values |
| `SUMMARY_PRECISION` | `4` | Fixed decimals for summary bounds |
| `SKIP_MISSING_CATALOGS` | `true` | Skip absent `_all.cat` files (count + report) |
| `SKIP_MALFORMED_ROWS` | `true` | Skip malformed rows (count + report) |

---

## FDConfig.hpp — FD Shear Test Parameters

All compile-time constants (rebuild required). Equivalent to the Fortran
measurement-program `para.inc`.

### Feature Feature switches

| Parameter | Default | Description |
|:---|:---:|:---|
| `FD_STATIC_MODE` | `PDF_SIGMA` | Statistical mode: `PDF_SIGMA` / `PDF_JACK` / `SWSE_JACK` |
| `FD_PER_EXPOSURE_STAR_BAR` | `false` | Star-bar mode: `false`=Nto1 global, `true`=NtoN per-exposure |

**Derived helpers** (compile-time, from `FD_STATIC_MODE`):
- `FD_USE_PDF_STATIS` - true for `PDF_SIGMA` or `PDF_JACK`
- `FD_USE_JACKKNIFE` - true for `PDF_JACK` or `SWSE_JACK`
- `FD_USE_SWSE_DATA` - true for `SWSE_JACK` only

### Dimensions Dimensions

| Parameter | Default | Description |
|:---|:---:|:---|
| `nmax_per_core` | `20000000` | Max sources per MPI node |
| `fd_num` | `21` | Spatial bins by field distortion |
| `PDF_BINS` | `4` | Equal-probability inner bins |
| `gf_lim` | `0.0015` | Spatial bin range (±gf_lim) |
| `NMAX` | `200` | Fine grid sampling points |
| `MAX_DUP` | `5` | Max duplicate measurements |

### Jackknife Jackknife / K-means

| Parameter | Default | Description |
|:---|:---:|:---|
| `N_jack` | `50` | Jackknife regions |
| `nmax_total` | `1000000` | Max total sources for k-means |
| `Km_iter` | `100` | K-means iterations |

### Quality Quality-cut thresholds

| Parameter | Default | Description |
|:---|:---:|:---|
| `snrfcut` | `4.0` | Fourier SNR cut |
| `starcut` | `20.0` | Star size cut |
| `chi2_thresh` | `0.01` | Exposure chi2 threshold |
| `flagcut` | `0.0` | Quality flag cut |
| `imaxcut` | `64.0` | Peak x cut |
| `jmaxcut` | `64.0` | Peak y cut |
| `zplow` | `0.0` | Min photometric redshift |
| `zphigh` | `3.0` | Max photometric redshift |
| `star_bar_mltp` | `3.0` | Star-bar sigma multiplier |
| `psf_chi2_mltp` | `3.0` | PSF chi2 sigma multiplier |

### External External-catalog cuts

| Parameter | Default | Description |
|:---|:---:|:---|
| `ft_cut` | `-1.0` | Flags_ft cut (<0 = skip) |
| `fg_cut` | `-10.0` | Flags_fg cut |
| `gold_cut` | `-10.0` | Flags_gold cut |
| `ext_cut` | `-4.0` | Ext_mash cut |

### Star-cut histogram Star-cut histogram parameters

| Parameter | Default | Description |
|:---|:---:|:---|
| `n_size_bins` | `100` | Size histogram bins |
| `n_mag_bins` | `20` | Magnitude histogram bins |
| `size_min` / `size_max` | `-2.0` / `2.0` | Size range |
| `mag_min_val` / `mag_max_val` | `10.0` / `30.0` | Magnitude range |
| `min_bin_count` | `100` | Min count per bin |
| `peak_match_tol` | `0.05` | Peak match tolerance |
| `min_concentration` | `0.6` | Min stellar locus concentration |
| `star_phy_min` / `star_phy_max` | `-0.5` / `0.2` | Physical star size range |

### Per-exposure star-cut Per-exposure star-cut parameters

| Parameter | Default | Description |
|:---|:---:|:---|
| `stage1_snr` | `40.0` | SNR for histogram accumulation |
| `stage2_snr` | `0.0` | Stage-2 SNR (0=disabled) |
| `init_win_active` | `0.1` | Initial window (active) |
| `init_win_fallback` | `0.15` | Initial window (fallback) |
| `default_s_init` | `0.5` | Default initial S_cut |
| `clip_nsigma` | `3.0` | Clipping sigma |
| `min_clip_limit` | `0.015` | Min clip limit |
| `default_s_std` | `0.05` | Default S_cut std |
| `fallback_scut_default` | `0.6` | Fallback S_cut |

### Catalog column Catalog column indices (0-based, DES format)

External-catalog columns (0-based):

| Index | Field |
|:---:|:---|
| 0 | `col_flags_ft` |
| 1 | `col_flags_fg` |
| 2 | `col_flags_gold` |
| 3 | `col_ext_mash` |
| 4 | `col_cra` (RA) |
| 5 | `col_cdec` (Dec) |
| 6/7 | `col_mag_g` (g band, even/odd mag+err) |
| 8/9 | `col_mag_r` |
| 10/11 | `col_mag_i` |
| 12/13 | `col_mag_z` |
| 14/15 | `col_mag_y` |
| 16 | `col_zp` (photometric redshift) |
| 18 | `col_ccd` (= `EXTCAT_TOTAL_COLUMNS`) |

Per-source columns are derived: `ccd_num_cols * ext_cat + LensingConfig::index`.

### Bad CCD Bad CCD list and chip-edge masking (DES)

| Parameter | Default | Description |
|:---|:---:|:---|
| `bad_ccds[]` | `{2, 31, 53, 61}` | Hard-coded bad CCD numbers |
| `n_bad_ccds` | `4` | Bad CCD count |
| `chip_xmin` / `chip_xmax` | `50` / `1990` | Chip x-edge mask range |
| `chip_ymin` / `chip_ymax` | `100` / `3990` | Chip y-edge mask range |

---

## Derived-width invariant

`process_rearr` does not always assume 18 external fields at runtime. With explicit external-catalog projection, its effective external width is the projection-list length; the complete row width is `external_width + 1 CCD column + 25 process_main fields`. Keep `ExtCatConfig`, `ExternalCatalogReader`, `ProcessRearrConfig`, and downstream FD column assumptions consistent when changing the schema.

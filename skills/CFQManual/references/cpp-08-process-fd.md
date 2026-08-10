# process_fd: FD (Field-Distortion) Shear Test

## Purpose

`process_fd` is the C++ equivalent of the Fortran measurement program. It reads
per-exposure shear catalogs (`_all.cat`), removes point sources (stars) via
star-bar fitting, bins sources by field distortion, and recovers the mean shear
per bin using either a PDF χ² sign test or a jackknife estimator. The output
(`FD_test_comb.dat`) is used for multiplicative/additive bias calibration.

This is the fifth and final pipeline phase, running after `process_rearr`.

## Source Files

| File | Role |
|:---|:---|
| `src/process_fd/process_fd.cpp` | Orchestration |
| `src/process_fd/ShearCatalogReader.cpp` | Read `_all.cat` catalogs |
| `src/process_fd/StarCutCalculator.cpp` | Star-bar point-source removal |
| `src/process_fd/KMeansClusterer.cpp` | Spherical k-means for jackknife regions |
| `src/process_fd/FDMeasurement.cpp` | Spatial binning + shear recovery |
| `src/process_fd/QuadraticFitting.cpp` | χ² minimization (quadratic fit) |
| `include/process_fd/*.hpp` | Headers |
| `include/process_fd/FDData.hpp` | In-memory data struct (replaces Fortran common blocks) |
| `config/FDConfig.hpp` | All parameters |

## Processing Flow

```
1. Load & broadcast exposure list
       │
2. Read shear catalogs (distributed across MPI ranks)
       │  ShearCatalogReader::readExposure
       │  -> parse rows, apply quality cuts, deduplicate, append to FDData
       ▼
3. Star-bar point-source removal
       │  StarCutCalculator (single global or per-exposure)
       │  -> build size-magnitude histogram
       │  -> locate stellar locus
       │  -> compute size cut threshold
       │  -> remove stars
       ▼
4. K-means jackknife region clustering (jackknife modes only)
       │  KMeansClusterer::runMPI
       │  -> spherical k-means on RA/Dec -> N_jack cluster centers
       │  -> assign each source to nearest cluster
       ▼
5. Shear recovery (per spatial bin, per component g1/g2)
       │  FDMeasurement::plotComparison
       │  -> bin sources by field distortion gf
       │  -> equal-probability inner binning
       │  -> χ² sign test (PDF modes) or ratio estimator (SWSE mode)
       │  -> quadratic fitting for best-fit c and sigma
       ▼
6. Write FD_test_comb.dat
```

## Data Structure

`FDData` (in `FDData.hpp`) collects all per-source arrays in one struct,
replacing Fortran common blocks:

| Field | Type | Description |
|:---|:---|:---|
| `x1, x2` | `vector<float>` | gf1, gf2 (field distortion) |
| `y1, y2` | `vector<float>` | g1, g2 (galaxy shear) |
| `de1, de2` | `vector<float>` | de∓h1, de±h1 (response, PDF mode) |
| `ww` | `vector<float>` | Jackknife weight |
| `magg, magr, magi` | `vector<float>` | Magnitudes |
| `sizerel` | `vector<float>` | Relative size |
| `src_snr` | `vector<float>` | Source SNR |
| `delta_chi2` | `vector<float>` | Stage 7 normalized PSF-minus-extended residual difference |
| `orth_ext` | `vector<float>` | Stage 7 PSF-orthogonal extension projection |
| `rra, ddec` | `vector<float>` | RA, Dec (degrees) |
| `iexpo` | `vector<int>` | Exposure index (per-exposure mode) |
| `snrf` | `vector<float>` | SNR_F (per-exposure mode) |
| `labels` | `vector<int>` | Jackknife region assignment |
| `ng` | `int` | Source count |

## Statistical Modes

`FD_STATIC_MODE` (in `FDConfig.hpp`) selects how mean shear (c_best) and
uncertainty (sigma) are estimated:

| Mode | c_best | sigma | Uses jackknife? | Uses k-means? |
|:---|:---|:---|:---:|:---:|
| `PDF_SIGMA` (default) | χ² sign test + quadratic fit | from PDF | no | no |
| `PDF_JACK` | χ² sign test | jackknife | yes | yes |
| `SWSE_JACK` | SWSE ratio estimator | jackknife | yes | yes |

**Derived compile-time flags:**
- `FD_USE_PDF_STATIS` = true for `PDF_SIGMA` or `PDF_JACK`
- `FD_USE_JACKKNIFE` = true for `PDF_JACK` or `SWSE_JACK`
- `FD_USE_SWSE_DATA` = true for `SWSE_JACK` only

## Star-Bar Modes

`FD_PER_EXPOSURE_STAR_BAR` (in `FDConfig.hpp`) selects point-source removal:

| Mode | Description |
|:---|:---|
| `false` (default) | Single global star bar (Nto1 / DES style): one `S_cut` for all exposures |
| `true` | Per-exposure star bar (NtoN / HSC style): one `S_cut` per exposure |

### Star-cut algorithm

1. Build a size-magnitude histogram (`n_size_bins` × `n_mag_bins`)
2. Locate the stellar locus (peak matching with `peak_match_tol`,
   `min_concentration`)
3. Compute the size cut threshold `S_cut`
4. Per-exposure mode: iterate per exposure with `stage1_snr`, `clip_nsigma`,
   `init_win_active/fallback`
5. Apply the cut: remove sources with size < `S_cut`

### Stage 7 point-source statistics are calibration inputs

`ShearCatalogReader` copies `col_delta_chi2` and `col_orth_ext` into `FDData`
for offline threshold calibration. The current `process_fd` implementation
does not use either field in `StarCutCalculator` and applies no fixed
`delta_chi2`/`orth_ext` selection. Point-source removal therefore remains the
existing star-bar size-magnitude procedure unless a separately calibrated cut
is deliberately implemented. `gal_size_T` and `psf_size_T` remain present in
the 48-column input row but are not copied into dedicated `FDData` arrays.

## Spatial Binning

Sources are binned by field distortion value (gf1 for g1 component, gf2 for g2):
- `fd_num = 21` spatial bins
- Range: ±`gf_lim` = ±0.0015
- Within each spatial bin: `PDF_BINS = 4` equal-probability inner bins
- Inner bin boundaries are iteratively adjusted for equal source count

## Shear Recovery

### PDF mode (`FD_USE_PDF_STATIS`)

For each spatial bin, for each trial mean shear `c`:
1. Compute residual: `r_j = y_j - c * de_j`
2. Bin into equal-probability inner bins
3. χ² sign test: count positive/negative residuals per inner bin
4. $\chi^2(c) = \sum_b \frac{(N_+^{(b)} - N_-^{(b)})^2}{2 N^{(b)}}$

Then minimize:
1. Coarse interval search over `c`
2. Fine grid sampling (`NMAX = 200` points)
3. Quadratic fit: $\chi^2(c) = a_1 c^2 + a_2 c + a_3$
4. Best-fit: $c_{\text{best}} = -a_2 / (2a_1)$, $\sigma_c = 1/\sqrt{2a_1}$

### Jackknife mode (`FD_USE_JACKKNIFE`)

- K-means clustering: `N_jack = 50` spherical regions
- Each source assigned to nearest cluster (max dot product on unit sphere)
- Jackknife: leave one cluster out, recompute, estimate variance

### SWSE mode (`FD_USE_SWSE_DATA`)

- Ratio estimator for `c_best`
- Jackknife for `sigma`

## Output

`FD_test_comb.dat` (written by rank 0 to the FD output directory):

```
Selected_NUM1  g1(FD)  g1(GAL)  sigma1  Selected_NUM2  g2(FD)  g2(GAL)  sigma2
```

One row per spatial bin (`fd_num = 21` rows):
- `Selected_NUM`: source count in the bin
- `g(FD)`: field-distortion shear (the "known input" proxy)
- `g(GAL)`: recovered galaxy shear (c_best)
- `sigma`: uncertainty (sigma_c)

### Output path resolution

| CLI option | Config default | Purpose |
|:---|:---|:---|
| `--fd-output-dir` | `"fdout"` | Output directory name |
| `--fd-output-base` | `""` | Base directory (empty = dataset root) |
| `--fd-expo-list` | `""` | Expo-list override (else derived from rearr list) |

If `--fd-output-base` is absolute, use it; otherwise it's the dataset root.
If `--fd-output-dir` is relative, join to base; if empty, defaults to base.

## Configuration

All parameters in `FDConfig.hpp` (compile-time). See
[cpp-03c-rearr-fd-parameters.md](cpp-03c-rearr-fd-parameters.md).

### Key parameters

| Parameter | Default | Description |
|:---|:---:|:---|
| `FD_STATIC_MODE` | `PDF_SIGMA` | Statistical mode |
| `FD_PER_EXPOSURE_STAR_BAR` | `false` | Star-bar mode |
| `fd_num` | `21` | Spatial bins |
| `PDF_BINS` | `4` | Equal-probability inner bins |
| `gf_lim` | `0.0015` | Spatial bin range |
| `NMAX` | `200` | Fine grid sampling points |
| `N_jack` | `50` | Jackknife regions |
| `nmax_per_core` | `20000000` | Max sources per MPI node |
| `snrfcut` | `4.0` | Fourier SNR cut |
| `starcut` | `20.0` | Star size cut |
| `chi2_thresh` | `0.01` | Exposure chi2 threshold |
| `bad_ccds[]` | `{2, 31, 53, 61}` | Hard-coded bad CCDs |
| `chip_xmin/xmax` | `50/1990` | Chip x-edge mask |
| `chip_ymin/ymax` | `100/3990` | Chip y-edge mask |

## Catalog Column Indices

FD uses absolute 0-based column indices derived from `LensingConfig`:

- External catalog columns (0-17): `col_flags_ft`, `col_flags_fg`,
  `col_flags_gold`, `col_ext_mash`, `col_cra` (RA), `col_cdec` (Dec),
  `col_mag_g/r/i/z/y`, `col_zp`
- `col_ccd` = `EXTCAT_TOTAL_COLUMNS` (18)
- Per-source columns: `ccd_num_cols * ext_cat + LensingConfig::index`
  (e.g., `col_g1 = 19 * 1 + 16 = 35`)
- New Stage 7 morphology columns in the default 48-column schema:
  `col_galsizeT=43`, `col_psfsizeT=44`, `col_delta_chi2=45`,
  `col_orth_ext=46`; appended exposure `col_chi2=47`

`FDConfig::ICHI2` is the row count (`48` by default), not the zero-based
exposure-chi2 index. Compile-time assertions require `delta_chi2` to follow
`psf_size_T`, `orth_ext` to follow `delta_chi2`, and `Chi2` to end the row.

## Relationship to the Fortran Measurement Program

`process_fd` is the C++ successor to the Fortran measurement program
documented in the `fqmethod` skill (measurement-01 through measurement-06). The
scientific algorithm (χ² sign test, equal-probability binning, quadratic
fitting, jackknife) is identical. Key differences:

- **No Python driver**: The Fortran program used a Python driver
  (`control.py`) for parameter-grid search. The C++ version is self-contained;
  parameter changes require editing `FDConfig.hpp` and recompiling.
- **Compile-time mode selection**: `FD_STATIC_MODE` and
  `FD_PER_EXPOSURE_STAR_BAR` are compile-time switches, not runtime options.
- **Integrated**: `process_fd` runs as a pipeline phase, not a separate program.

# process_main Scientific & Numerical Parameters

These values live primarily in `config/LensingConfig.hpp`. Most are compile-time and require recompilation. One important exception is the mutable `LensingConfig::SOURCE_CAT`: `main.cpp` assigns it from `RuntimeOptions::extcat_output_directory`, so `--extcat-output` changes the effective source-catalog tile path at runtime. In `cpp_Lite`, eight Standard branch selectors were removed from the code and cannot be re-enabled by merely adding the constant back.

## LensingConfig.hpp - Scientific & Numerical Constants

This header contains the main scientific/numerical controls. Treat `constexpr` symbols as compile-time (rebuild required); treat mutable `SOURCE_CAT` separately as described above. The numerical constants correspond broadly to the legacy Fortran parameter blocks.

### 4a. Stage control

| Parameter | Default | Description |
|:---|:---:|:---|
| `PROCESS_stage` | `223092870` | Stage bitmask (product of primes 2·3·5·7·11·13·17·19·23) |
| `ASTROMETRY_trivial` | `0` (Std only; Lite=0) | 0=Gaia astrometry, 1=trivial (identity) |
| `include_FLAT` | `0` (Std only; Lite=0) | 0=no super-flat, 1=enable using `FLAT_PATH` |
| `include_Mask` | `2` (Std only; Lite=2) | 0=none, 1=legacy, 2=per-chip DQ mask, 3=legacy+DQ |

### 4b. Image / CCD size

| Parameter | Default | Description |
|:---|:---:|:---|
| `npx` | `3000` | Nominal CCD width (px) |
| `npy` | `5000` | Nominal CCD height (px) |

There is no `strl` path limit in the live C++ main pipeline. The former
`strl=150` declaration was unused and has been removed. Main paths use
`std::string`; their practical limits come from the filesystem and the selected
I/O library. The separate initializer compatibility guard defaults to 150 and
is configured with `--f77-max-path` (see
[cpp-05-process-init.md](cpp-05-process-init.md)).

### 4c. Split and background

| Parameter | Default | Description |
|:---|:---:|:---|
| `ext_cat` | `1` (Std; Lite=1) | 0=disable external catalog, 1=enable `SOURCE_CAT` |
| `ext_PSF` | `0` (Std; Lite=0) | 0=measure PSF from frame stars, 1=external PSF image |
| `CCD_split` | `2` | 1=whole-chip amp, 2=two-amplifier split |
| `blocksize` | `200` | Block size for background estimation (px) |
| `nct` | `12` | Number of rectangle blocks |
| `ncx` | `3` | Number of blocks in x |

### 4d. PSF selection

| Parameter | Default | Description |
|:---|:---:|:---|
| `psf_order` | `8` | PSF polynomial order |
| `npo` | `64` | PSF sample count |
| `npox` | `8` | PSF samples in x |
| `nstar_min` | `96` | Min stars for PSF fit (`npo * 3 / 2`) |
| `npl` | `10` | Polynomial degree for local PSF |
| `nplx` | `2` | Local PSF x-degree |
| `nstar_min_local` | `16` | Min true stars per chip (local mode) |
| `step_psf` | `100` (Std only) | PSF star step |
| `deblending` | `1` | 0=no deblending, 1=enabled |
| `n_neighbor` | `5` (Std only) | Neighbor count for deblending |
| `PSF_type` | `1` | 1=local polynomial, 2=hybrid (Std only) |
| `PSF_Ms` | `0` | 0=local PSF only, 1=enable PCA (Std only) |

### 4e. Stamp dimensions

| Parameter | Default | Description |
|:---|:---:|:---|
| `ns` | `64` | Stamp / power spectrum size (px) |
| `chip_margin` | `8` | Chip edge margin |
| `nl_2` | `40` | Half stamp + margin (`ns/2 + chip_margin`) |
| `nl` | `80` | Full stamp with margin (`nl_2 * 2`) |
| `flag_thresh` | `3` | Flag threshold |
| `dz_thresh` | `0.1` | Redshift threshold for catalog matching |

### 4f. Catalog sizes and limits

| Parameter | Default | Description |
|:---|:---:|:---|
| `ngal_max` | `4000` | Max galaxies per chip |
| `nstar_max` | `2000` | Max stars per chip |
| `src_npara` | `12` | Shared source/PSF/FFT2 internal row width through `iSNR_F` |
| `shear_cat_ncols` | `28` | Stage 7 row width through `iorth_ext`; excludes exposure chi2 |
| `expo_cat_ncols` | `29` | Exposure-catalog width: 28 Stage 7 fields plus exposure chi2 |
| `npd` | `33` | PU polynomial distortion terms |
| `NMAX_EXPO` | `25000` | Max exposures per run |
| `NMAX_CHIP` | `62` | Max CCDs per exposure |

### 4g. Thresholds

| Parameter | Default | Description |
|:---|:---:|:---|
| `source_thresh` | `2.0` | Source detection SNR threshold |
| `core_thresh` | `4.0` | Core detection threshold |
| `flat_thresh` | `0.01` (Std only) | Flat threshold |
| `area_max` | `4096` | Max source area (`ns * ns`) |
| `area_thresh` | `6` | Min source area |
| `SNR_PSF` | `100.0` | PSF star SNR threshold |
| `saturation_thresh` | `25000.0` | Saturation threshold |

### 4h. Smoothing

| Parameter | Default (Std) | Description |
|:---|:---:|:---|
| `gal_smooth` | `0` (Std) / `0` (Lite) | Galaxy smoothing parameter |
| `star_smooth` | `2` | Star smoothing parameter |

### 4i. Stage 7 Fourier morphology measurements

All values below are compile-time constants shared by Standard and Lite.
`point_stat_eps` and `point_stat_min_corr` are numerical validity guards, not
science-selection thresholds.

| Parameter | Default | Description |
|:---|:---:|:---|
| `size_fit_rmax` | `4` | Radius in Fourier pixels of the equal-weight low-frequency curvature fit that produces `gal_size_T` and `psf_size_T` |
| `point_stat_beta` | `0.10` | Fixed survey-wide shape of the extended template $P(k)\exp[-\beta k^2/k_{\max}^2]$; not fitted per source |
| `point_stat_k_frac` | `0.90` | Fraction of the PSF reliable radius used as $k_{\max}$ for `delta_chi2` and `orth_ext` |
| `point_stat_eps` | `1.0e-20` | Positive floor for singularity rejection and scalar normalization |
| `point_stat_min_corr` | `1.0e-6` | Minimum absolute normalized source-PSF projection required for a valid `orth_ext` |

### 4j. Noise-sigma estimation (`Set_Sig` mode-bar estimator)

These constants mirror the Fortran `sig_para.inc`. They control the robust
mode-bar noise-plane estimator: block seeds -> sky mode -> lower-side width ->
symmetric clipped plane fits -> validation -> normalization.

| Parameter | Default | Description |
|:---|:---:|:---|
| `sig_blocksize` | `200` | Block size for noise estimation |
| `sig_max_blocks` | `2048` | Max blocks sampled |
| `sig_min_block_pixels` | `1000` | Min pixels per block |
| `sig_min_block_triples` | `1000` | Min valid triples per block |
| `sig_min_blocks` | `4` | Min valid blocks for a fit |
| `sig_hist_nbin` | `256` | Histogram bins for mode finding |
| `sig_hist_range` | `6.0` | Histogram range (sigma) |
| `sig_min_mode_count` | `500` | Min count for mode |
| `sig_min_lower_count` | `1000` | Min count for lower-side width |
| `sig_lower_quantile` | `0.3173105` | Lower quantile for width |
| `sig_clip_k` | `3.0` | Clipping sigma |
| `sig_clip_niter` | `2` | Clipping iterations |
| `sig_min_fit_triples` | `1000` | Min triples for plane fit |
| `sig_min_fit_frac` | `0.20` | Min fraction for fit |
| `sig_median_ratio` | `1.2678405` | Median-to-sigma ratio |
| `sig_plane_min` | `1.0e-8` | Min plane value |
| `sig_max_plane_ratio` | `4.0` | Max plane variation ratio |
| `sig_pivot_min` | `1.0e-8` | Min pivot for plane solve |
| `sig_scale_s1` | `0.673475` | Stage-1 calibration candidate |
| `sig_scale_s2` | `1.027786` | Stage-2 calibration (active) |
| `sig_scale` | `1.027786` | Active selector (=`sig_scale_s2`) |

### 4k. Standard-only PCA/multi-scale PSF parameters

Compiled only by `cpp_Standard`, active only when `PSF_Ms=1`. Absent in Lite.

| Parameter | Default | Description |
|:---|:---:|:---|
| `rescale_size` | `1.2` | Target PSF size for residual rescaling |
| `procs_pn` | `40` | Process-group size for PCA scheduling |
| `work_pn` | `10` | Concurrent workers in PCA grouping |
| `nblocks` | `2` | Spatial blocks per CCD axis (2×2 grid) |
| `n_pcs` | `100` | Max PCA principal components |
| `npp6th` | `28` | 2D sixth-degree polynomial terms for PCA surfaces |
| `pca_negative_eigenvalue_threshold` | `-1.0e-5` | Invalid-eigenvalue threshold |
| `nmax_star_pchip` | `1000000` | Reserved legacy PCA star capacity (unused) |

### 4l. File-system paths

`SOURCE_CAT` is special: the header supplies the initial value, but the unified driver overwrites it from the runtime extcat output path before phases execute. `ASTROMETRY_CAT`, `FLAT_PATH`, and `PSF_PATH` are not exposed through `RuntimeOptions` in the current driver.

| Parameter | CLI | Default | Description |
|:---|:---|:---|:---|
| `SOURCE_CAT` | `--extcat-output` | `/lustre/.../des_y6_cat` | External source-catalog tile directory |
| `ASTROMETRY_CAT` | - | `/lustre/.../gaia_cat_sorted` | Gaia reference tiles directory |
| `FLAT_PATH` | - | `/lustre/.../DES_super_flat/i2014` (Std only) | Per-chip flat FITS files |
| `PSF_PATH` | - | `"hahahaha"` (Std only) | External PSF image directory |

### 4m. Internal catalog column indices (0-based)

Zero-based positions in the per-source result rows. Changing them changes the
internal/output layout and requires coordinated reader/writer changes.

| Parameter | Default | Field |
|:---|:---:|:---|
| `iid` | `0` | Historical index name; Stage 7 overwrites this slot with normalized PSF-model `poly_chi2` |
| `ipixx` | `1` | Source-center pixel x (`xc`) |
| `ipixy` | `2` | Source-center pixel y (`yc`) |
| `isig` | `3` | Local source noise sigma |
| `istar` | `4` | Number of PSF stars available for the chip (`nstar`) |
| `i_imax` | `5` | Peak x |
| `i_jmax` | `6` | Peak y |
| `ih_flux` | `7` | Half-light flux |
| `ih_area` | `8` | Source area |
| `iflag` | `9` | Quality flag |
| `iPSF` | `10` | Local PSF FWHM |
| `iSNR_F` | `11` | Fourier S/N |
| `ira` | `12` | Right ascension |
| `idec` | `13` | Declination |
| `igf1` | `14` | Field distortion 1 |
| `igf2` | `15` | Field distortion 2 |
| `ig1` | `16` | Shear estimator 1 |
| `ig2` | `17` | Shear estimator 2 |
| `ide` | `18` | Normalization estimator |
| `ih1` | `19` | Higher-order 1 |
| `ih2` | `20` | Higher-order 2 |
| `icos2` | `21` | Spin-2 cosine |
| `isin2` | `22` | Spin-2 sine |
| `iparity` | `23` | WCS parity |
| `igalsizeT` | `24` | Galaxy low-frequency Fourier-power curvature size $T$ (pixel²) |
| `ipsfsizeT` | `25` | Local-PSF low-frequency Fourier-power curvature size $T$ (pixel²) |
| `idelta_chi2` | `26` | Normalized PSF-minus-extended residual difference |
| `iorth_ext` | `27` | PSF-orthogonal extension projection |
| `ichi2` | `28` | Exposure chi2 appended after the 28-field Stage 7 row (index 28, count 29) |

### 4n. Calibration and camera geometry

| Parameter | Default | Description |
|:---|:---:|:---|
| `g1_c` | `-0.001` | Additive correction for field-distortion 1 |
| `g2_c` | `-0.0003` | Additive correction for field-distortion 2 |
| `chi2_thresh` | `0.01` | Max exposure/PSF chi2 in catalog combination |
| `chipnx` | `2046` | Nominal science CCD width (PSF normalization) |
| `chipny` | `4094` | Nominal science CCD height (PSF normalization) |
| `pixel_size` | `0.2628` | DECam pixel scale (arcsec) |

---

## Important non-config scientific literals

Not every tunable value is in a config header. In the current source, `ShearMeasurement::getShear` defines `float PSFr_ratio = 0.75f;` locally in `src/process_main/ShearMeasurement.cpp`. Changing the Fourier_Quad PSF-radius window ratio therefore requires editing that implementation and rebuilding; it is **not** a `LensingConfig` parameter. Before changing any value that is not found in a config header, search the selected variant for the exact symbol/literal and inspect its call site.

# process_main Detailed Stage Reference

## Stage 7: Shear Measurement (Core)

| | |
|:---|:---|
| **Prime** | 17 |
| **Function** | `ShearMeasurement::procShear(int iexpo)` |
| **Files** | `src/process_main/ShearMeasurement.cpp`, `include/process_main/ShearMeasurement.hpp` |
| **Config** | `LensingConfig.hpp`: `ns`, `pixel_size`; implementation-local `PSFr_ratio=0.75f` in `ShearMeasurement::getShear` |

### What it does

For each galaxy in each chip:
1. Read the galaxy noise-subtracted power spectrum (Stage 6)
2. Evaluate the local PSF model (Stage 5) at the galaxy position
3. Compute PSF quality diagnostics (poly_chi2, FWHM)
4. Compute celestial coordinates (RA, Dec) and field-distortion (gf1, gf2,
   rotation, parity) via PU mapping
5. **Fourier_Quad estimator**: deconvolve the PSF, compute 5 shear estimators
6. Rotate estimators from pixel to celestial coordinates
7. Write the 24-column shear catalog row

### Key sub-functions

| Function | Purpose |
|:---|:---|
| `procShear` | Stage 7 main entry |
| `expoShear` | Per-exposure driver |
| `getShear` | Fourier_Quad estimator: compute g1, g2, de, h1, h2 |
| `getWindowMinK` / `getWindowMinKVer2` | Compute the low-k cutoff radius |
| `getPSFArea` | Compute PSF FWHM |

### The 5 shear estimators

From the PSF-deconvolved, Gaussian-windowed galaxy power spectrum
$M(\mathbf{k}) = W(\mathbf{k}) G(\mathbf{k})$:

- **g1, g2** - shear signal (quadrupole anisotropy)
- **de** - isotropic response (normalization denominator)
- **h1, h2** - fourth-order anisotropy (calibration correction)

For the full mathematical derivation, see the `fqmethod` skill's
`pipeline-07-shear-estimation.md`.

### Coordinate rotation

Estimators are rotated from pixel to celestial frame using the field
distortion Jacobian's rotation angle φ and parity:
- Rank-2 (g1, g2): rotate by 2φ
- Rank-4 (h1, h2): rotate by 4φ
- Parity correction: flip second-component signs if parity = -1

### Output row (24 columns, 0-based)

`iid, ipixx, ipixy, isig, istar, i_imax, i_jmax, ih_flux, ih_area, iflag,
iPSF, iSNR_F, ira, idec, igf1, igf2, ig1, ig2, ide, ih1, ih2, icos2, isin2,
iparity`

(Stage 8 appends `ichi2` as the 25th field.)

---

## Implementation-local tuning note

`PSFr_ratio` is currently a local literal (`float PSFr_ratio = 0.75f;`) inside `ShearMeasurement::getShear`, not a `LensingConfig` symbol. Edit the implementation (or deliberately promote it into config) and rebuild if this window ratio must change.

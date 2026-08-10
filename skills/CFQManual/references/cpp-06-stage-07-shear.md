# process_main Detailed Stage Reference

## Stage 7: Shear and Fourier-Power Morphology Measurement (Core)

| | |
|:---|:---|
| **Prime** | 17 |
| **Function** | `ShearMeasurement::procShear(int iexpo)` |
| **Files** | `src/process_main/ShearMeasurement.cpp`, `src/process_main/CurvatureSizeMeasurement.cpp`, `src/process_main/PointSourceStatistics.cpp`, and matching headers |
| **Config** | `LensingConfig.hpp`: `ns`, `pixel_size`, `size_fit_rmax`, `point_stat_*`; implementation-local `PSFr_ratio=0.75f` in `ShearMeasurement::getShear` |

The curvature-size and point-source-statistic implementations are identical in
Standard and Lite. Their Stage 7 integration differs only where Standard keeps
optional PSF-model branches that Lite has physically removed.

### Processing flow

For each galaxy in each chip:

1. Read the Stage 6 noise-subtracted galaxy power spectrum.
2. Evaluate the Stage 5 local PSF power model at the galaxy position.
3. Measure `gal_size_T` from the galaxy power and `psf_size_T` from the local
   PSF power.
4. Measure the PSF-versus-extended-template statistics `delta_chi2` and
   `orth_ext` from the same galaxy and PSF powers.
5. Compute PSF diagnostics (`poly_chi2`, FWHM).
6. Compute RA/Dec, field distortion (`gf1`, `gf2`), rotation, and parity from
   the PU astrometric mapping.
7. Deconvolve the PSF and compute the five Fourier_Quad estimators `g1`, `g2`,
   `de`, `h1`, and `h2`.
8. Rotate the spin-2 and spin-4 estimators into celestial coordinates.
9. Write one 28-column Stage 7 row.

### Key functions and data

| Symbol | Purpose |
|:---|:---|
| `procShear` | Stage 7 exposure entry point |
| `expoShear` | Per-exposure/chip driver and catalog writer |
| `measurePowerCurvatureSize` | Low-frequency 2-moment curvature-size fit |
| `PointSourceStatistics::measure` | Joint source/PSF inner products and two morphology statistics |
| `PointSourceStatisticsResult` | In-memory `delta_chi2`, `orth_ext`, and validity flag; invalid values remain `-999` |
| `getShear` | Fourier_Quad `g1`, `g2`, `de`, `h1`, `h2` estimator |
| `getWindowMinK` / `getWindowMinKVer2` | Fourier cutoff helpers used by the shear estimator |
| `getPSFArea` | PSF FWHM measurement |

## Low-frequency curvature size (`gal_size_T`, `psf_size_T`)

`measurePowerCurvatureSize(ns, power, rmax)` uses the centered Fourier disk
$u^2+v^2\le r_{\max}^2$. With equal weight for every finite pixel, it fits

$$P(x)=B_0+B_1x+B_2x^2,\qquad
x=\frac{u^2+v^2}{r_{\max}^2}.$$

Define

$$q_{\max}^2=\left(\frac{2\pi}{n_s}\right)^2r_{\max}^2.$$

The published curvature size is

$$T=-\frac{2B_1}{B_0q_{\max}^2}.$$

The default `size_fit_rmax=4` is a compile-time Fourier-pixel radius. Stage 7
applies the same fit independently to the galaxy power (`gal_size_T`) and the
local PSF model (`psf_size_T`). Both outputs are in pixel².

The function writes `-999` when dimensions/radius are invalid, the requested
disk is incomplete, fewer than six finite pixels remain, the normal equations
are singular, $B_0\le0$, or the resulting $T$ is non-finite, non-positive, or
outside the `float` range. There is no separate validity column.

## Fourier-power point-source morphology statistics

These statistics compare the measured source power $M(k)$ with the local PSF
power $P(k)$ and one fixed extended template. They provide observables for
point-source calibration; they do **not** apply a calibrated source-selection
cut inside Stage 7 or `process_fd`.

### Reliable Fourier window and taper

1. Find the global positive PSF peak $P_{\max}$.
2. Let $r_{\rm win}$ be the nearest Fourier radius whose PSF power is not
   greater than $10^{-4}P_{\max}$ (non-finite values also bound the window).
3. Set $k_{\max}=\mathtt{point\_stat\_k\_frac}\,r_{\rm win}$; the default
   fraction is `0.90`.
4. Exclude the DC mode and use only finite source values with finite positive
   PSF power at $0<k<k_{\max}$.

The radial weight is unity through $0.8k_{\max}$ and then uses a raised-cosine
taper:

$$w(k)=\frac12\left[1+\cos\left(\pi
\frac{k/k_{\max}-0.8}{0.2}\right)\right],
\qquad 0.8k_{\max}<k<k_{\max}.$$

### Fixed extended template and inner products

The single survey-wide template is

$$T(k)=P(k)\exp\left[-\beta\frac{k^2}{k_{\max}^2}\right],
\qquad \beta=\mathtt{point\_stat\_beta}=0.10.$$

`beta` is a compile-time template shape, not a per-source fitted parameter.
Using the tapered inner product
$\langle A,B\rangle=\sum_k w(k)A(k)B(k)$, the implementation accumulates

$$mm=\langle M,M\rangle,\quad mp=\langle M,P\rangle,\quad
pp=\langle P,P\rangle,$$

$$mt=\langle M,T\rangle,\quad tt=\langle T,T\rangle,\quad
pt=\langle P,T\rangle.$$

### `delta_chi2`

Define the best-amplitude residuals for the PSF and extended hypotheses:

$$R_P=\max\left(0,mm-\frac{mp^2}{pp}\right),\qquad
R_T=\max\left(0,mm-\frac{mt^2}{tt}\right).$$

Then

$$\mathtt{delta\_chi2}=\frac{R_P-R_T}{mm+\epsilon}.$$

Smaller or negative values are more PSF-like; positive values favor the fixed
extended template. This is a normalized residual difference, not the exposure
`Chi2` appended later by Stage 9.

### `orth_ext`

Remove from the extended template the component parallel to the PSF, then
measure the normalized source projection along that orthogonal direction:

$$\mathtt{orth\_ext}=
\frac{mt-pt\,mp/pp}{\sqrt{\max(0,tt-pt^2/pp)}}
\frac{\sqrt{pp}}{|mp|+\epsilon}.$$

An ideal PSF-like source is near zero; larger positive values indicate more
extension in the fixed-template direction. Both published statistics are
designed to be invariant to an overall source-brightness scale.

### Validity rules

Both values remain `-999` and the in-memory `valid` flag remains false when the
input shape is wrong, the PSF has no usable reliable-window boundary, fewer
than eight valid Fourier pixels remain, a norm/projection is singular or
non-finite, the absolute normalized source-PSF correlation is below
`point_stat_min_corr=1e-6`, or a result cannot be represented as `float`.
`point_stat_eps=1e-20` and `point_stat_min_corr` are numerical protections, not
survey science cuts. Negative finite noise-subtracted source power is allowed;
the per-mode PSF power must be positive.

## Fourier_Quad shear estimators and rotation

From the PSF-deconvolved, Gaussian-windowed galaxy power
$W(\mathbf{k})G(\mathbf{k})$, Stage 7 computes:

- `g1`, `g2`: quadrupole shear signal;
- `de`: isotropic response;
- `h1`, `h2`: fourth-order calibration terms.

For the full estimator derivation, see the `fqmethod` skill's
`pipeline-07-shear-estimation.md`. The field-distortion Jacobian rotates
`g1/g2` by $2\phi$ and `h1/h2` by $4\phi$; parity `-1` flips each second
component.

## Catalog contract

### Stage 7 `*_shear.dat` row (28 columns, 0-based)

The exact text-header order is:

`poly_chi2, xc, yc, sigma, nstar, imax, jmax, half_light_flux,
half_light_area, flag, psf_FWHM, SNR_F, ra, dec, gf1, gf2, g1, g2, de, h1,
h2, cos2, sin2, parity, gal_size_T, psf_size_T, delta_chi2, orth_ext`

The corresponding `LensingConfig` indices are `iid` through `iorth_ext`;
`iid` is a historical name and Stage 7 stores normalized `poly_chi2` in that
slot. Stage 8 computes exposure diagnostics; Stage 9
appends that exposure's `Chi2` at zero-based `ichi2=28`, producing a 29-field
process-main payload. With the default 18 external fields and one CCD field,
the final `_all.cat` width is `18 + 1 + 29 = 48`.

`process_fd` resolves all four new absolute columns. It currently stores
`delta_chi2` and `orth_ext` in `FDData` for offline threshold calibration but
does not apply an uncalibrated fixed cut. The curvature sizes remain in the
catalog schema even though the current FD reader does not copy them into
dedicated arrays.

## Focused validation

Both variants expose:

```bash
make test-point-source-statistics
```

The synthetic regression covers PSF-like and fixed-beta extended sources,
brightness scaling, negative noise-subtracted power, invalid inputs, and the
28/29-column compile-time contract. It directly tests the point-source module;
curvature-size behavior is covered by source review/build integration rather
than a separate focused test target.

## Tuning boundary

Changing `size_fit_rmax` or any `point_stat_*` value requires rebuilding the
selected variant. Calibrate `point_stat_beta`, `point_stat_k_frac`, and any
science-selection threshold on representative labeled data. Do not repurpose
the numerical guards as science cuts. `PSFr_ratio` remains a separate local
literal (`0.75f`) inside `ShearMeasurement::getShear`.

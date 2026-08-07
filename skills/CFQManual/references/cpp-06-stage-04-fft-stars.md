# process_main Detailed Stage Reference

## Stage 4: FFT Stage 1 (Stars)

| | |
|:---|:---|
| **Prime** | 7 |
| **Function** | `FourierTransformSt1::procFourierTSt1(int iexpo)` |
| **Files** | `src/process_main/FourierTransformSt1.cpp`, `include/process_main/FourierTransformSt1.hpp` |
| **Config** | `LensingConfig.hpp`: `ns`, `ext_PSF` |

### What it does

For each star candidate:
1. Extract a `ns × ns` stamp centered on the star
2. Compute the 2D FFT power spectrum
3. Subtract noise power
4. Regularize (suppress high-frequency noise)
5. Write the star power spectrum for Stage 5 PSF modeling

In Standard, `ext_PSF=1` bypasses the normal star-power path (`FourierTransformSt1` contains an early-return branch). The external PSF image itself is consumed later by Stage 7 (`ShearMeasurement`) from `PSF_PATH/PSF.fits`; do not implement external-PSF changes here without tracing that Stage 7 branch.

---

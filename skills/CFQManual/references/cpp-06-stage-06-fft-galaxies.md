# process_main Detailed Stage Reference

## Stage 6: FFT Stage 2 (Galaxies)

| | |
|:---|:---|
| **Prime** | 13 |
| **Function** | `FourierTransformSt2::procFourierTSt2(int iexpo)` |
| **Files** | `src/process_main/FourierTransformSt2.cpp`, `include/process_main/FourierTransformSt2.hpp` |
| **Config** | `LensingConfig.hpp`: `ns`, `gal_smooth` |

### What it does

For each galaxy:
1. Extract a `ns × ns` stamp centered on the galaxy
2. Compute the 2D FFT power spectrum
3. Subtract noise power
4. Regularize
5. Apply optional galaxy smoothing (`gal_smooth`)
6. Write the galaxy power spectrum for Stage 7

---

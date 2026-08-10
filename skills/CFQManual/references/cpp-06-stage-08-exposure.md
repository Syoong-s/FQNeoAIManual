# process_main Detailed Stage Reference

## Stage 8: Exposure Info

| | |
|:---|:---|
| **Prime** | 19 |
| **Function** | `ExposureInfo::procInfo(int iexpo)` |
| **Files** | `src/process_main/ExposureInfo.cpp`, `include/process_main/ExposureInfo.hpp` |
| **Config** | `LensingConfig.hpp`: `NMAX_EXPO`, `chi2_thresh` |

### What it does

For each chip:
1. Aggregate per-chip diagnostics (N-valid-chip, PSF-FWHM, chi_d-stars,
   nstar-per-chip, cRVAL1, cRVAL2)
2. Store in the global `expo_para` array (`6 × NMAX_EXPO`)

After all ranks finish:
3. `MPI_Allreduce` (sum) across all ranks
4. Rank 0 writes `expo_info.dat`:
   ```
   N-valid-chip PSF-FWHM(arcsec) chi_d-stars nstar-per-chip cRVAL1 cRVAL2 expo_name
   ```
5. Stage 9 appends this exposure chi2 as the 29th process-main field
   (`ichi2=28`, zero-based) after the 28-field Stage 7 row

### Key data

| Variable | Type | Description |
|:---|:---|:---|
| `ExposureInfo::expo_para` | `vector<float>` | Global array `6 × NMAX_EXPO` (reduced across ranks) |
| `expo_info.dat` | output | Per-exposure diagnostics (written by rank 0) |

---

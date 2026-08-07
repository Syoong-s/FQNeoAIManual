# process_main Detailed Stage Reference

## Stage 1: Pre-processing

| | |
|:---|:---|
| **Prime** | 2 |
| **Function** | `PreProcess::preProcess(int iexpo)` |
| **Files** | `src/process_main/PreProcess.cpp`, `include/process_main/PreProcess.hpp` |
| **Config** | `LensingConfig.hpp`: `CCD_split`, `blocksize`, `nct`, `ncx`, `include_Mask`, `include_FLAT`, `sig_*` |

### What it does

For each chip in the exposure:
1. Read the Science image and (if `include_Mask`) the DQ mask
2. Estimate and subtract the background (`setBackground`)
3. Apply optional super-flat (`include_FLAT=1`, Standard only)
4. Normalize noise to a uniform sigma plane (`setSig`)
5. Build a defect/weight map (`locateDefects`, `mergeDefects`, `maskSourceRegions`)
6. Detect stripes, stellar halos, and dents
7. Match Gaia stars for astrometry (Gaia branch only)
8. Write `Norm/`, `cat_Orig/`, weight maps

### Key sub-functions

| Function | Purpose |
|:---|:---|
| `chipPreProcess` | Per-chip driver |
| `setBackground` | Block-based background estimation |
| `flattenChip` | Flatten the chip background |
| `setSig` | Estimate/validate/apply noise-sigma plane (the mode-bar estimator) |
| `locateDefects` | Find defective pixels |
| `mergeDefects` | Merge defect maps |
| `detectStripes` / `detectStellarHalo` / `detectDent` | Detect artifacts |

### The `Set_Sig` mode-bar noise estimator

This is the F6 mode-bar estimator (mirrors Fortran `sig_para.inc`). It:
1. Obtains robust block seeds
2. Finds the sky mode and lower-side width
3. Performs two symmetric clipped plane fits
4. Validates the final plane (`sig_max_plane_ratio`, `sig_plane_min`)
5. Normalizes the amplifier immediately using `sig_scale`

The active scale is `sig_scale = sig_scale_s2 = 1.027786` (Stage-2 calibration).

---

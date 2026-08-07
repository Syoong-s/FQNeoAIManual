# process_main Detailed Stage Reference

## Stage 2: Astrometry

| | |
|:---|:---|
| **Prime** | 3 |
| **Function** | `Astrometry::procAstrometry(int iexpo)` |
| **Files** | `src/process_main/Astrometry.cpp`, `include/process_main/Astrometry.hpp` |
| **Config** | `LensingConfig.hpp`: `npd`, `ASTROMETRY_trivial`, `ASTROMETRY_CAT` |

### What it does

For each chip:
1. Load pre-processed image and weight map
2. Match Gaia reference stars (from `ASTROMETRY_CAT` tiles)
3. Fit the PU (polynomial distortion) astrometric solution
4. Write astrometry data (`dat_Astro/<P>_astro.dat`), WCS headers (`Head/`),
   and check data (`dat_Chk/`)

### Key sub-functions

| Function | Purpose |
|:---|:---|
| `chipProcessAstrometry` | Per-chip driver |
| `coordinateTransferPU` | PU polynomial pixel <-> sky transformation |
| `fieldDistortionPU` | Compute field distortion shear (gf1, gf2, rotation, parity) |
| `genAstrometryData` | Gaia-based astrometric solution (Standard + Lite) |
| `genAstrometryDataTrivial` | Trivial (identity) astrometry (Standard only, `ASTROMETRY_trivial=1`) |
| `getRaDecRangeFine` | Fine RA/Dec range with PU distortion |

The PU polynomial uses `npd = 33` distortion terms. `fieldDistortionPU` computes
the field distortion shear `(gf1, gf2)` and rotation `(cos2, sin2)` used by
Stage 7 for coordinate rotation and by `process_fd` for spatial binning.

---

# process_main Detailed Stage Reference

## Stage 5: PSF Modeling

| | |
|:---|:---|
| **Prime** | 11 |
| **Function** | `PSFModel::procPSF(int iexpo)` |
| **Files** | `src/process_main/PSFModel.cpp`, `include/process_main/PSFModel.hpp` |
| **Config** | `LensingConfig.hpp`: `PSF_type`, `PSF_Ms`, `psf_order`, `npo`, `npl`, `nstar_min`, `nstar_min_local`, `n_pcs`, `npp6th`, `nblocks` |

### What it does

Two sub-modes based on `PSF_Ms`:

#### `PSF_Ms=0` (default, both Standard and Lite): Local polynomial PSF

1. Select PSF stars via χ² clustering (quality + size)
2. Fit a spatial polynomial PSF model per chip (`makePSFLocalFit`)
3. Evaluate the PSF at each galaxy position in Stage 7

#### `PSF_Ms=1` (Standard only): Multi-scale / PCA PSF

1. Local polynomial fit (same as above)
2. `PSFRecons::chipPSFRecons(N_EXPO)` - PCA residual reconstruction:
   - Load PCA components and mean PSF (`initAndLoadAllPSF`)
   - Fit PCA coefficient surfaces (2D sixth-degree polynomial, `npp6th=28`)
   - Reconstruct the residual PSF model
3. Free PCA memory (`freePSFMemory`)

### Key sub-functions

| Function | Purpose |
|:---|:---|
| `procPSF` | Stage 5 main entry |
| `initAndLoadAllPSF` | Load/broadcast PCA components (Standard only) |
| `freePSFMemory` | Free PCA storage (Standard only) |
| `itpNormPSF` | Normalized polynomial PSF fit |
| `getPSFModel` | Evaluate polynomial PSF at (x, y) |
| `getPSFModelVeryLocal` | Very-local PSF interpolation (Standard only, `PSF_type=2`) |
| `getPowerAll` | Compute PSF ellipticity and size |
| `PSF_rescale` / `PSF_unscale` | Rescale PSF residuals (Standard only) |

### `PSF_type` options (Standard only)

- `1` (default): Local polynomial fit (`makePSFLocalFit`)
- `2`: Hybrid PSF (polynomial + very-local interpolation, `interpolatePSF`)

Lite implements only `PSF_type=1`.

---

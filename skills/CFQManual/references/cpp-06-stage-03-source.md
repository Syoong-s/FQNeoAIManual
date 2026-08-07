# process_main Detailed Stage Reference

## Stage 3: Source Detection

| | |
|:---|:---|
| **Prime** | 5 |
| **Function** | `SourceExtractor::procSource(int iexpo)` |
| **Files** | `src/process_main/SourceExtractor.cpp`, `include/process_main/SourceExtractor.hpp` |
| **Config** | `LensingConfig.hpp`: `source_thresh`, `core_thresh`, `deblending`, `ext_cat`, `n_neighbor` |

### What it does

For each chip:
1. Detect sources above `source_thresh` (SNR)
2. Deblend overlapping sources (`deblending=1`)
3. Classify star candidates (size-magnitude locus)
4. If `ext_cat=1`: match detected sources to the external catalog (RA/Dec)
5. Build per-chip source catalogs and star candidate lists

### Key sub-functions

| Function | Purpose |
|:---|:---|
| `chipProcessSource` | Per-chip driver |
| `deBlending` | Separate overlapping sources |
| `genSourceExtCatalog` | Build source catalog from external catalog match |
| `genSourceCatalog` | Build source catalog without external match (Standard only) |
| `genStarCandidate` / `genStarCandidateDirect` | Identify star candidates |
| `findNoise` / `checkSource` | Validate detections |
| `generateGalCatFileName` | Derive catalog filename from RA/Dec |

---

# Parameter Router: Choose the Smallest Detailed Reference

Use this file only to decide which parameter document to load. Do not load every parameter reference unless the task spans multiple configuration domains.

| User task | Load |
|---|---|
| Phase switches, CLI options, input/output roots, external catalog parser, dataset defaults | [cpp-03a-runtime-parameters.md](cpp-03a-runtime-parameters.md) |
| `process_main` scientific/numerical constants, stage bitmask, PSF, stamps, thresholds, noise estimator, column indices | [cpp-03b-main-parameters.md](cpp-03b-main-parameters.md) |
| Rearrangement layout or FD-test statistics/cuts/binning | [cpp-03c-rearr-fd-parameters.md](cpp-03c-rearr-fd-parameters.md) |
| User describes an outcome and asks which parameter(s) to change | [cpp-03d-tuning-guide.md](cpp-03d-tuning-guide.md), then the relevant detailed file above |

## Runtime versus compile-time rule

There is **no single precedence chain across all configuration headers**. `main.cpp` constructs `ProcessConfig::RuntimeOptions` from compiled defaults and CLI options override only fields represented in that struct. Scientific constants such as `LensingConfig::PROCESS_stage`, `ns`, PSF controls, and most `FDConfig` values remain compile-time and require a rebuild. Always check whether a requested symbol is a `RuntimeOptions` field before promising a no-rebuild change.

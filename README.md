# CFQManual

CFQManual is a selectively loaded background-manual plugin for **Fourier Quad Pipeline Neo (C++)**. It is designed so an AI can understand the pipeline structure and behavior, choose and tune parameters, generate production runner files or lightweight directly runnable HPC Slurm jobs, debug execution, and make targeted source changes without loading the complete manual into context.

## Design

- **Thin top-level index**: `SKILL.md` routes tasks instead of duplicating the full manual.
- **Selective loading**: parameters are split by configuration domain and `process_main` is split by numerical stage.
- **Detailed leaves**: routed references retain implementation-level files, symbols, algorithms, outputs, constraints, and modification notes.
- **Runtime/build-time distinction**: CLI/`RuntimeOptions` settings are separated from compile-time scientific constants and implementation-local literals.
- **Stage 7 morphology contract**: low-frequency curvature sizes and Fourier-power point-source statistics are documented through their formulas, validity rules, catalog columns, and downstream consumers.
- **Two HPC modes**: production runner generation remains available, while direct-HPC mode can generate a compact Slurm job that acquires/builds a SIF, binds requested data/source paths, compiles if needed, and launches the pipeline with `srun`.
- **Runner templates**: Docker `.env`, production `cpppipeline.env`, and direct-HPC Slurm/env templates are bundled with field-level generation rules.
- **Standard/Lite awareness**: Lite's deleted branches are treated as deleted code, not merely disabled constants.

## Plugin layout

```text
CFQManual-plugin-1.3.0/
├── .agents/plugins/marketplace.json
├── .codex-plugin/plugin.json
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── skills/CFQManual/
│   ├── SKILL.md
│   ├── agents/openai.yaml
│   ├── references/
│   └── templates/
├── CHANGELOG.md
├── README.md
├── README.zh-CN.md
└── LICENSE
```

CFQManual is a knowledge/routing skill and therefore does not install lifecycle hooks.

## Install from GitHub marketplace

### Codex

```bash
codex plugin marketplace add Syoong-s/FQNeoAIManual
codex plugin add CFQManual@CFQManual-plugin
```

### Claude Code

```text
/plugin marketplace add Syoong-s/FQNeoAIManual
/plugin install CFQManual@CFQManual-plugin
```

## Install from an extracted release

### Codex

```bash
codex plugin marketplace add /absolute/path/to/CFQManual-plugin-1.3.0
codex plugin add CFQManual@CFQManual-plugin
```

### Claude Code

```bash
claude --plugin-dir /absolute/path/to/CFQManual-plugin-1.3.0
```

## Invocation

The skill name is `CFQManual`. In Codex it can be invoked explicitly as `$CFQManual`. It also permits implicit invocation when the task clearly concerns Fourier Quad Pipeline Neo (C++).

## Automatic releases

`.github/workflows/release.yml` runs when a tag matching `v*` is pushed. For example:

```bash
git tag -a v1.3.0 -m "Release v1.3.0"
git push origin v1.3.0
```

The workflow validates that the tag version matches the Codex and Claude manifests, then packages the complete `.agents/`, `.claude-plugin/`, `.codex-plugin/`, and `skills/` trees together with README, CHANGELOG, and LICENSE. It publishes ZIP and tar.gz archives plus SHA256 checksums to the GitHub Release.

Repository: https://github.com/Syoong-s/FQNeoAIManual

## License

MIT. See `LICENSE`.

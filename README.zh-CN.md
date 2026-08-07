# CFQManual

CFQManual 是面向 **Fourier Quad Pipeline Neo (C++)** 的选择性加载背景知识插件。目标是让没有项目背景的 AI 在不一次性加载全部知识的情况下，仍能理解 Pipeline 的代码结构与行为、准确选择/调整参数、生成生产级 runner 或轻量可直接提交的 HPC Slurm 作业、排查运行问题，并按开发需求精准修改源码。

## 设计原则

- **顶层索引尽量小**：`SKILL.md` 只负责路由，不重复整份手册。
- **按任务选择性加载**：参数文档按配置域拆分，`process_main` 按 9 个数值 stage 拆分。
- **叶子文档保持详细**：对应 reference 保留文件、namespace/函数、参数、算法流程、输入输出、耦合约束和修改注意事项。
- **区分运行时与编译期**：只有进入 `RuntimeOptions` 的字段才能被 CLI 覆盖；大量科学参数仍需修改头文件并重新编译，部分值直接写在实现函数内。
- **双 HPC 部署模式**：既保留生产级 runner，也支持轻量 direct-HPC 模式，可生成获取/构建 SIF、挂载目录、必要时编译并通过 `srun` 直接运行 Pipeline 的 Slurm。
- **自带 runner 模板**：包含 Docker `.env`、生产 `cpppipeline.env`、direct-HPC env/Slurm 模板和字段级生成规则。
- **正确处理 Standard/Lite**：Lite 中被冻结的 8 条分支是被物理删除，而不是简单设置不同常量。

## 插件结构

```text
CFQManual-plugin-1.2.1/
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

CFQManual 是知识/路由型 skill，不依赖会话生命周期，因此没有加入不必要的 hooks。

## 通过 GitHub marketplace 安装

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

## 从解压后的 Release 安装

### Codex

```bash
codex plugin marketplace add /absolute/path/to/CFQManual-plugin-1.2.1
codex plugin add CFQManual@CFQManual-plugin
```

### Claude Code

```bash
claude --plugin-dir /absolute/path/to/CFQManual-plugin-1.2.1
```

## 使用

Skill 名称为 `CFQManual`。在 Codex 中可以显式调用 `$CFQManual`；当请求明确涉及 Fourier Quad Pipeline Neo (C++) 时，也允许宿主隐式启用该 skill。

## 自动 Release

`.github/workflows/release.yml` 在推送 `v*` tag 时自动运行。例如：

```bash
git tag -a v1.2.1 -m "Release v1.2.1"
git push origin v1.2.1
```

工作流会先检查 tag 版本与 Codex/Claude manifest 版本一致，再完整打包 `.agents/`、`.claude-plugin/`、`.codex-plugin/` 和 `skills/`，同时加入 README、CHANGELOG、LICENSE，生成 ZIP、tar.gz 和 SHA256，并发布到 GitHub Release。

仓库：https://github.com/Syoong-s/FQNeoAIManual

## License

MIT，见 `LICENSE`。

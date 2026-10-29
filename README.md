# deepseek-harness-source-analysis

DeepSeek Harness（`dsh`）源码解析笔记。第三方非官方项目。

本仓库：https://github.com/luzhiming/deepseek-harness-source-analysis

## 上游是什么

DeepSeek 官方的开源 agent harness，TypeScript 写的，架构是「一切皆插件」，底层框架是 [Cordis](https://github.com/cordiverse/cordis)。上游处于开发者预览期，随时可能有 breaking change。

- 仓库：https://github.com/deepseek-ai/deepseek-harness
- 官网：https://deepseek.com/harness
- 文档：https://deepseek-harness.github.io/deepseek-harness/ （英文 `/en/`）
- 跑起来：`npx @deepseek-ai/dsh web` → http://127.0.0.1:3080
- 论文（Cordis 设计）：https://arxiv.org/abs/2608.25512
- 讨论区：https://github.com/deepseek-ai/deepseek-harness/discussions
- Discord：https://discord.gg/4MrtZUhpxg

上游协议：MIT，`Copyright (c) 2026 DeepSeek`。

## 笔记索引

- [01 Fiber 的 epoch、依赖链指纹与 uid](./01-fiber-epoch-and-uid.md) —— Fiber 用 `epoch` 记录「当前依赖由哪些提供者 Fiber（uid）组成」；依赖变了就卸再载，没变就不动。

## 协议

本仓库 MIT，完整条款见 [LICENSE](./LICENSE)，`Copyright (c) 2026 luzhiming`。

上游 DeepSeek Harness 同样是 MIT（`Copyright (c) 2026 DeepSeek`），第三方依赖见官方 [THIRD_PARTY_NOTICES.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/THIRD_PARTY_NOTICES.md)。本仓库对上游代码只做分析性引用（路径、行号、短片段）。

「DeepSeek」「dsh」等商标归 DeepSeek AI 所有。本仓库非官方，与 DeepSeek AI 无隶属或背书关系。

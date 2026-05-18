# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指导。

## 项目概述

TencentDB Agent Memory 是一个面向 AI 代理（OpenClaw/Hermes）的四层记忆系统插件，提供：

- **L0**：原始对话捕获（JSONL）
- **L1**：结构化记忆提取（LLM 驱动）
- **L2**：场景/情景聚合
- **L3**：用户画像生成

系统通过符号化短期记忆（Mermaid 画布）和分层长期记忆（画像/场景）将 token 消耗降低约 61%。

## 构建与开发命令

```bash
# 安装依赖（postinstall 自动运行 after-tool-call patch）
npm install

# 构建全部（插件 + 迁移脚本）
npm run build

# 运行测试
npm test                    # 单次运行
npm run test:watch          # 监听模式
npm run test:coverage       # 带覆盖率报告

# OpenClaw 集成（在仓库根目录执行）
openclaw plugins install --link .   # 链接为本地插件
openclaw gateway restart             # 重启以加载变更

# 注意：开发时无需构建步骤 - OpenClaw 直接在运行时加载 .ts 文件
```

## 架构

```
index.ts                    # OpenClaw 插件入口 - 注册钩子、转换事件
├── src/
│   ├── core/
│   │   ├── tdai-core.ts    # 宿主无关的记忆门面（OpenClaw 和 Hermes 共用）
│   │   ├── types.ts        # 抽象接口（HostAdapter、LLMRunner、RuntimeContext）
│   │   ├── store/          # 存储层（SQLite + sqlite-vec、TCVDB）
│   │   ├── conversation/   # L0 捕获
│   │   ├── record/         # L1 提取
│   │   ├── scene/          # L2 场景聚合
│   │   ├── persona/        # L3 画像生成
│   │   ├── hooks/          # 自动召回和自动捕获逻辑
│   │   ├── prompts/        # LLM 提示词模板
│   │   └── tools/          # tdai_memory_search、tdai_conversation_search
│   ├── adapters/
│   │   ├── openclaw/       # OpenClawHostAdapter（进程内）
│   │   └── standalone/     # StandaloneLLMRunner（HTTP/Gateway）
│   ├── offload/           # 短期压缩（Mermaid 画布 + 卸载）
│   ├── gateway/           # 独立 HTTP Gateway 服务器
│   └── utils/             # Pipeline 管理器、检查点、记忆清理器
├── hermes-plugin/          # Hermes 代理的 Python 适配器（独立于 src/）
└── scripts/               # 迁移工具、CLI 辅助脚本
```

## 关键模式

- **TdaiCore + HostAdapter**：核心记忆逻辑与宿主无关。OpenClaw 使用 `OpenClawHostAdapter`，Gateway 使用 `StandaloneHostAdapter`。
- **记忆分层**：钻取链路始终为 L3 → L2 → L1 → L0，通过 `result_ref` 链接。
- **Pipeline 调度**：`MemoryPipelineManager` 编排 L1/L2/L3 提取，包含预热、空闲超时和间隔逻辑。
- **Mermaid 画布**：短期记忆使用 Mermaid 语法进行紧凑的状态表示，通过 `node_id` 实现可追溯性。
- **配置层级**：三层调优级别（日常、高级、完整），预设合理默认值——零配置开箱即用。

## 依赖

- **运行时**：Node.js >= 22.16.0
- **可选依赖**：OpenClaw >= 2026.3.7、node-llama-cpp >= 3.16.2
- **存储**：SQLite + sqlite-vec（本地）、腾讯云向量数据库（云端）
- **Embedding**：通过 node-llama-cpp 本地化，或远程 OpenAI 兼容 API

## 数据目录

记忆产物位于 `~/.openclaw/memory-tdai/`：

- `L0/` - 原始对话 JSONL
- `L1/` - 提取的记忆记录
- `L2/` - 场景块（Markdown）
- `L3/` - 画像文件（`persona.md`）

## 配置

完整 schema 参见 `openclaw.plugin.json`。关键路径：

- `src/config.ts` - 配置解析和默认值
- `README.md` - 用户级配置指南（三层调优）

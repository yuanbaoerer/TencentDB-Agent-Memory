# TencentDB Agent Memory 记忆系统设计分析

## 概览

TencentDB Agent Memory 的核心设计不是把历史对话切片后丢进一个扁平向量库，而是把记忆拆成两条互补链路：

- 长期个性化记忆：`L0 Conversation -> L1 Atom -> L2 Scenario -> L3 Persona`
- 短期任务记忆：工具日志 offload -> 摘要 JSONL -> Mermaid 任务图 -> 上下文压缩与注入

这两条链路共同服务同一个目标：让 Agent 既能保留可追溯证据，又能在上下文里看到高密度、可操作的结构化信息。

长期记忆强调跨 session 的偏好、事实、指令和场景沉淀；短期任务记忆强调长任务中的工具结果压缩、任务状态恢复和 token 控制。

## 长期记忆分层

### L0 Conversation：原始对话层

L0 是证据层，负责保存原始用户/助手对话。

关键实现：

- `src/core/conversation/l0-recorder.ts`
- `src/core/hooks/auto-capture.ts`

L0 的数据写入 `conversations/YYYY-MM-DD.jsonl`。每条记录包含 `sessionKey`、`sessionId`、角色、内容、时间戳和 message id。

这一层有几个重要细节：

- 使用增量游标避免重复捕获历史消息。
- 使用 `originalUserText` 替换被召回上下文污染过的用户输入。
- 对内容做 sanitize，避免注入标签、工具噪音和过长内容污染后续抽取。
- 同步或异步写入检索索引，支持后续 L0 原文搜索。

L0 的定位是“可恢复、可追溯、可检索的事实来源”，不是直接面向 Agent 的高层记忆。

### L1 Atom：原子结构化记忆

L1 从 L0 中提取结构化原子记忆。

关键实现：

- `src/core/record/l1-extractor.ts`
- `src/core/record/l1-writer.ts`
- `src/core/record/l1-dedup.ts`

L1 使用 LLM 从对话消息中提取三类记忆：

- `persona`：用户偏好、长期特征、表达习惯。
- `episodic`：历史事件、任务过程、时间相关事实。
- `instruction`：用户明确要求、规则、约束、SOP。

每条 L1 记忆包含：

- `content`
- `type`
- `priority`
- `scene_name`
- `source_message_ids`
- `metadata`
- `timestamps`
- `sessionKey`
- `sessionId`

写入策略是双写：

- JSONL：`records/YYYY-MM-DD.jsonl`，作为 append-only 的可恢复记录。
- VectorStore/FTS：作为实时检索引擎。

去重逻辑采用两阶段：

1. 先用向量搜索或 FTS 找候选相似记忆。
2. 再由 LLM 批量判断 `store / update / merge / skip`。

如果向量不可用，会退化到 FTS；如果 FTS 也不可用，则跳过去重直接写入。这让系统在 embedding 或数据库异常时仍能继续工作。

### L2 Scenario：场景块

L2 把零散 L1 原子记忆聚合成可读的场景 Markdown。

关键实现：

- `src/core/scene/scene-extractor.ts`
- `src/core/scene/scene-index.ts`
- `src/core/scene/scene-navigation.ts`

L2 的产物主要在：

- `scene_blocks/*.md`
- scene index metadata
- persona 文件尾部的 scene navigation

SceneExtractor 会把新增 L1 记忆交给带工具能力的 LLM，让它在 `scene_blocks/` 沙箱内创建、更新、合并或软删除场景文件。

这一层的设计重点是把记忆从“事实列表”提升为“事件/主题结构”。它还维护场景数量上限，接近上限时提示 LLM 优先合并或更新已有场景，避免场景无限膨胀。

### L3 Persona：用户画像层

L3 是最高层的稳定用户画像。

关键实现：

- `src/core/persona/persona-generator.ts`
- `src/core/persona/persona-trigger.ts`

L3 产物是 `persona.md`。它从 L2 场景中抽取长期稳定的用户偏好、工作方式、约束和重要背景。

触发条件包括：

- 首次场景提取完成。
- persona 文件缺失或损坏后的恢复。
- 距离上次生成后累计新增记忆达到阈值。
- L2 明确发出 persona update request。

PersonaGenerator 会优先读取自上次 persona 生成后变化过的场景，做增量更新，而不是每次全量扫描所有历史。

## 运行链路

统一入口是 `src/core/tdai-core.ts`。它通过 `HostAdapter` 和 `LLMRunnerFactory` 抽象宿主环境，使核心记忆逻辑可以运行在 OpenClaw、Hermes、Gateway 或 CLI 中。

一次典型交互链路如下：

1. 对话结束后，宿主调用 `handleTurnCommitted()`。
2. `performAutoCapture()` 写入 L0 JSONL，并写入 L0 检索索引。
3. `MemoryPipelineManager.notifyConversation()` 更新 session 状态。
4. Pipeline 根据阈值、空闲时间或 flush 触发 L1。
5. L1 完成后调度 L2。
6. L2 完成后按条件触发 L3。
7. 下一轮开始前，宿主调用 `handleBeforeRecall()`。
8. `performAutoRecall()` 搜索 L1，读取 L3 persona 和 L2 scene navigation，并注入上下文。

## Pipeline 调度

Pipeline 调度器位于 `src/utils/pipeline-manager.ts`。

它的核心职责是把捕获和抽取解耦：capture 阶段只负责可靠记录，是否做 L1/L2/L3 由调度器决定。

L1 触发路径：

- 会话轮数达到阈值。
- session 空闲超过 `l1IdleTimeoutSeconds`。
- session end 或进程 shutdown 时 flush。

默认启用 warm-up：

```text
1 -> 2 -> 4 -> ... -> everyNConversations
```

这样新 session 的前几轮会更快被抽取，成熟 session 则降低抽取频率。

L2 采用 per-session downward-only timer：

- L1 完成后延迟触发 L2。
- 遵守最小间隔，避免频繁重算。
- 遵守最大间隔，保证活跃 session 最终会被聚合。
- session 超过活跃窗口后停止轮询。

L3 使用全局串行队列和 pending 标记，避免 persona 并发写入。

## 召回与注入

召回入口在 `src/core/hooks/auto-recall.ts`。

召回上下文被拆成两类：

- `prependContext`：动态 L1 相关记忆，放到用户 prompt 前。
- `appendSystemContext`：稳定的 persona、scene navigation 和工具指南，放到 system prompt 后。

这样做的原因是 L1 召回每轮变化较大，不适合放进 system prompt；L2/L3 变化较少，放进 system prompt 更有利于 prompt cache。

搜索策略支持：

- `keyword`：FTS5 BM25。
- `embedding`：向量相似度。
- `hybrid`：关键词和向量融合。SQLite 后端使用 RRF，TCVDB 后端可使用原生 hybrid search。

召回不仅注入片段，还会告诉 Agent 如何继续主动检索：

- `tdai_memory_search`：搜索 L1 结构化记忆。
- `tdai_conversation_search`：搜索 L0 原始对话。
- `read_file`：读取 scene navigation 中定位到的 Markdown 场景。

这体现了 progressive disclosure：先给高层结构和少量相关片段，不够时再向下钻取证据。

## 存储设计

存储抽象在 `src/core/store/types.ts`，后端选择在 `src/core/store/factory.ts`。

当前支持：

- `sqlite`：默认本地后端，使用 `vectors.db`，支持 sqlite-vec 和 FTS5。
- `tcvdb`：腾讯云向量数据库后端，支持服务端 embedding 和 hybrid search。

典型数据目录：

```text
~/.openclaw/memory-tdai/
├── conversations/      # L0 原始对话 JSONL
├── records/            # L1 原子记忆 JSONL
├── scene_blocks/       # L2 场景 Markdown
├── persona.md          # L3 用户画像
├── vectors.db          # SQLite 检索库
└── .metadata/          # checkpoint、manifest、scene index 等
```

存储接口是 capability-based。上层代码不直接依赖 SQLite 或 TCVDB，而是根据能力判断是否可用：

- vector search
- FTS search
- native hybrid search
- sparse vectors
- profile sync
- deferred embedding update

这让系统在不同后端之间保持同一套 pipeline 和 recall 逻辑。

## 短期 Context Offload

短期任务记忆位于 `src/offload/`。

它解决的是长任务中工具结果过多、上下文膨胀的问题，不等同于长期用户记忆。

主要层次：

- 原始工具结果：写入 `refs/*.md`。
- L1 摘要：写入 `offload-<session>.jsonl`。
- L2 Mermaid：把任务状态组织成 `mmds/*.md`。
- L3 压缩：在上下文接近阈值时，用摘要替换工具结果，必要时删除旧消息，并注入 Mermaid 任务图。

关键实现：

- `src/offload/hooks/after-tool-call.ts`
- `src/offload/hooks/before-prompt-build.ts`
- `src/offload/storage.ts`
- `src/offload/pipelines/l2-mermaid.ts`
- `src/offload/hooks/llm-input-l3.ts`

长期记忆关注“跨 session 学到什么”；短期 offload 关注“当前长任务如何不中断、不爆上下文”。

## 设计评价

这个项目的记忆系统更像一个分层记忆操作系统，而不是一个简单 RAG 插件。

主要优点：

- 可追溯：L3/L2 可以向下追到 L1/L0。
- 可调试：persona、scene blocks、JSONL 都是可读文件。
- 可降级：embedding 失败时可以走 FTS，召回超时时跳过注入。
- 可迁移：核心通过 HostAdapter 抽象宿主。
- 可控膨胀：L1 有去重，L2 有场景上限，L3 有增量更新。

主要代价：

- 链路复杂，涉及 LLM 抽取、调度、checkpoint、文件和数据库一致性。
- L2/L3 依赖带工具能力的 LLM 写文件，需要较强的沙箱和后处理保护。
- JSONL append-only 与实时 VectorStore 删除之间存在最终一致性，需要 cleaner 或恢复逻辑配合。
- 召回效果依赖 L1 抽取质量、embedding 配置、FTS 分词和 scene/persona 更新节奏。

总体来看，它的核心取舍是：牺牲一部分实现复杂度，换取“高层结构 + 底层证据”的可恢复、可审计记忆能力。

# TencentDB-Agent-Memory · 一页速览

## 解决什么问题

- 每开一个新会话，都要重新跟 coding agent 解释项目背景、SOP、工具约定、输出格式 —— 同样的话一遍遍重复，agent 跨会话「记不住」。
- 长会话里工具调用日志（grep 输出、文件读取、bash 回显）持续堆积，撑爆 context window —— token 暴涨变贵，还把任务相关信息挤出窗口导致任务失败。
- 现有两条退路都不行：暴力堆全部历史 → 又贵又超限；不可逆的 lossy summarization → 信息丢了，出错时无法回溯到原始细节。

## 用什么办法

> **记忆不该是平的：把扁平存储换成金字塔分层，把笨重的工具日志换成可下钻的符号画布。**

两套独立子系统，对付两种不同的「遗忘」。**长程记忆**把对话蒸馏成 L0 原始对话 → L1 原子事实 → L2 场景块 → L3 人格画像的金字塔，下次会话只注入顶层 persona，需要细节时才下钻。**短程记忆**把会话内的工具输出全量存进 `refs/*.md`，context 里只留一张 Mermaid 符号画布，出错时按 `node_id` 钻回原始日志。

输入：宿主 agent 的对话事件 + 工具调用事件
输出：`persona.md` + `scene_blocks/*.md`（跨会话注入）/ Mermaid `.mmd` 画布（会话内替换臃肿上下文）

---

## 实现架构

```
HOSTS      index.ts            gateway/server.ts        cli/index.ts
(entry)    OpenClaw 插件壳     Hermes HTTP sidecar      seed 命令
               │                     │                      │
ADAPTER   adapters/openclaw    adapters/standalone           │
               └──────────┬──────────┘                      │
                          ▼                                 │
CORE              core/tdai-core.ts  (TdaiCore facade) ◄─────┘
                          │
         ┌────────────────┴───────────────────┐
   长程轨道                              短程轨道
         │                                    │
  hooks/auto-capture                  offload/hooks/after-tool-call
         ▼                                    ▼
  conversation/l0-recorder  (L0)      offload/pipelines/l2-mermaid
         ▼                              (工具日志 → .mmd 符号画布)
  utils/pipeline-manager                      ▼
  (L0→L1→L2→L3 调度器)                 offload/hooks/llm-input-l3
   │      │       │                    (按分数替换/删除陈旧块)
   ▼      ▼       ▼                            │
  l1-    scene-  persona-                      ▼
  extractor extractor generator        offload/storage
  (L1)    (L2)    (L3)                  refs/*.md + mmds/*.mmd
   │      │       │
   └──────┴───┬───┘
              ▼
   core/store/  (IMemoryStore：SQLite | TCVDB | embedding)
              │
  hooks/auto-recall ──► 把 persona + 场景导航回注进 context
```

**4 层结构**

| 层 | 角色 | 模块 |
|---|---|---|
| Entry | 三种宿主入口 | `index.ts`、`gateway/server.ts`、`cli/index.ts` |
| Adapter | 把宿主 API 翻译成中立接口 | `adapters/openclaw`、`adapters/standalone` |
| Core | 中立门面 + 双轨流水线 | `core/tdai-core.ts`、`utils/pipeline-manager.ts`、`core/record`、`core/scene`、`core/persona`、`offload/` |
| Cross-cutting | 存储 / 沙箱 / 清洗 | `core/store`（IMemoryStore）、`utils/clean-context-runner.ts`、`utils/sanitize.ts` |

---

## 关键洞察

**1. 双轨记忆，因为遗忘有两种本质**
跨会话的痛是「重复解释」，会话内的痛是「上下文溢出」—— 二者解法不同。`core/` 处理长程分层金字塔，`offload/` 处理会话内符号压缩，两者只共享 `core/store` 抽象，互不耦合。

**2. host-neutral facade + adapter，把宿主差异挤到边缘**
`TdaiCore` 只依赖 `HostAdapter` / `LLMRunner` 抽象接口，从不认识具体宿主。OpenClaw（进程内）和 Hermes（HTTP sidecar）各写一个 adapter，同一套记忆逻辑就能嵌进两种环境。`index.ts` 因此只是个 thin shell —— 只做事件翻译和注册。

**3. 进展式披露，而非有损摘要**
工具输出全量留在 `refs/*.md`，context 里只放 Mermaid 顶层符号，出错时凭 `node_id` 下钻回原文。作者明确拒绝不可逆的 lossy summarization —— 压缩必须可回溯，这是 offload 轨道的设计红线。

**4. L1/L2/L3 不是同步调用链，是三种语义的定时器**
`pipeline-manager` 里：L1 是 idle-debounce（每轮对话重置倒计时），L2 是 downward-only timer（触发时间只能提前不能推后），L3 是 global-mutex（全局串行）。因为 LLM 抽取很贵，必须按会话冷热错峰触发，不能每轮都跑。

**5. 抽取交给被沙箱锁死的 LLM agent**
L2/L3 让 LLM 用工具自主读写场景文件，但 `workspaceDir` 钉死在 `scene_blocks/` —— `checkpoint`、`scene_index`、`persona.md` 对 LLM 物理不可见。给了自主权，又防它改坏系统文件。

**6. IMemoryStore 是 capability-based 抽象，且 fault-tolerant**
SQLite（本地）和腾讯云 VectorDB（云端）实现同一接口，上层 hooks/tools 零修改即可切换；向量/全文/混合检索表达为能力位，缺失时优雅降级。所有方法出错返回空而非抛异常 —— 记忆失败不该拖垮 agent。

**7. 注入是闭环的最后一步，不是副产品**
`auto-recall` 在 agent 启动前把 persona、场景导航、记忆工具指南回注进 context，并显式限制 agent 每轮最多搜索 3 次。沉淀下来的金字塔只有被「读回去」才有价值。

---

## 最短描述

> TencentDB-Agent-Memory 是 **coding agent 的双层记忆中间件**：长程对话被蒸馏成 L0→L1→L2→L3 人格金字塔、会话内工具日志被压成可按 `node_id` 下钻的 Mermaid 符号画布，通过 host-neutral 的 `TdaiCore` 门面同时嵌入 OpenClaw 插件与 Hermes HTTP sidecar —— 让 agent 跨会话不必被重复教，长任务里不被自己的日志撑爆。

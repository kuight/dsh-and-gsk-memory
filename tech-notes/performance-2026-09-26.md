# 性能实测：browser 侧边栏 vs node 直连（2026-09-26）

## 背景
用户在新 Chrome 用户资料加载 dsh 扩展，侧边栏发"你好"，感知首 token 约 23s、出字慢。做只读诊断 + node 直连对照。

## 1. 侧边栏会话实测（browser profile 3081，会话 session-3ce3499a…）
来自会话事件/projection cache（未改动任何配置）：

- **输入 token**：uncachedInputTokens = **10699**（contextPressure.pressureTokens 同值；surfaceTokens 2726）；contextWindow 1000000 → 压力 ~1.1%
- **内部首 token（ttft）**：ttftMs = **13397**（13.4s）
- **输出 token**：outputTokens / decodeTokens = **178**
- **每秒输出**：decodeMs = 23984 → **7.4 tok/s**（178/23.98）
- **总 LLM 耗时**：llmMs = 37381（37.4s）
- **重试**：llmRetry 空，无重试
- **标题生成请求**：`session/title-llm-request` 在 `+0.070s` 发出，主请求 request/context 在 `+0.067s` → **与主请求同时发出**（同 route：nvidia / deepseek-v4.1-flash）
- 时间线：turn/start +0.023s → request/header +0.066s → request/context +0.067s → title-llm-request +0.070s → assistant/message +37.444s（含 reasoning + text，186 字） → step/end + turn/end +37.445s completed

## 2. node 直连对照（绕开 dsh，不带工具，key 走 credentials，不打印）
- **NVIDIA** deepseek-ai/deepseek-v4.1-flash，请求 ~300 字回复：
  - 首 token **25620ms（25.6s）**，总耗时 70035ms，输出 338 字符（流式 delta 51 段）→ 约 **0.7 delta/s**
- **DEEPSEEK 官方**（api.deepseek.com/v1，DEEPSEEK_API_KEY）：**HTTP 401 invalid key**（api key ****a8a1 invalid）→ 不可用；计费方式：**待用户确认**（该 key 无效，无法实测算费）

## 结论/观察
- 侧边栏首 token 用户感知 ~23s ≈ dsh 内部 ttft 13.4s + 扩展/渲染开销；与 node 直连 25.6s 同量级（NVIDIA 端波动）。
- 首 token 慢主要来自 **输入 token 量大（~10.7k）→ 预填充耗时**，输出本身尚可（7.4 tok/s），但用户感知"出字慢"可能是流式渲染/SSE 从 deepseek-v4.1-flash 输出。
- DEEPSEEK_API_KEY 不可用，勿用它做 provider。

## 关联
- tech-notes/dsh-upgrade-0.1.5.md（升级与 NO_PROXY）
- tech-notes/profiles.md（扩展 ID、侧边栏数值）
- tech-notes/known-facts.md（延迟分档）
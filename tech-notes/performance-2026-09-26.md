# 性能实测：browser 侧边栏 vs node 直连（2026-09-26）

## 背景
用户在新 Chrome 用户资料加载 dsh 扩展，侧边栏发"你好"，感知首 token 约 23s、出字慢。做只读诊断 + node 直连对照。

## 1. 侧边栏会话实测（browser profile 3081，会话 session-3ce3499a…）
来自会话事件/projection cache（未改动任何配置）：

- **输入 token**：uncachedInputTokens = **10699**（contextPressure.pressureTokens 同值；surfaceTokens 2726）；contextWindow 1000000 → 压力 ~1.1%
- **内部首 token（TTFT）**：ttftMs = **13397**（13.4s）
- **输出 token**：outputTokens / decodeTokens = **178**
- **每秒输出**：decodeMs = 23984 → **7.4 tok/s**（178/23.98）
- **总 LLM 耗时**：llmMs = 37381（37.4s）
- **重试**：llmRetry 空，无重试
- **标题生成请求**：`session/title-llm-request` 在 `+0.070s` 发出，主请求 request/context 在 `+0.067s` → **与主请求同时发出**（同 route：nvidia / deepseek-v4.1-flash）
- 时间线：turn/start +0.023s → request/header +0.066s → request/context +0.067s → title-llm-request +0.070s → assistant/message +37.444s（含 reasoning + text，186 字） → step/end + turn/end +37.445s completed

## 2. node 直连对照（绕开 dsh，不带工具，key 走 credentials，不打印）
两次直连数据（均为 deepseek-ai/deepseek-v4.1-flash）：

| 时间 | 场景 | 首 token | 总耗时 | 输出 |
|---|---|---|---|---|
| 2026-09-26 上午 | 简短回复（"收到"） | **11.2s** | 11.5s | ~4 字 |
| 2026-09-26 晚间 | ~300 字回复 | **25.6s** | 70.0s | 338 字符（流式 51 段） |

- 两次直连首 token 差异显著（11.2s vs 25.6s），同模型同端点。
- **DEEPSEEK 官方**（api.deepseek.com/v1，DEEPSEEK_API_KEY，模型 deepseek-flash）：**HTTP 401 invalid key** → **key 失效**，该 provider 不可用；计费情况**待用户确认**。

## 结论（修正后）
- ❌ 旧结论"首 token 慢主要来自预填充（输入 token 大）"**不成立**——直连 25.6s 慢于侧边栏内部 13.4s，且两次直连波动大（11–26s），与输入 token 无关。
- ✅ **NVIDIA 免费端点延迟波动大**（直连首 token 11–26s），输出约 **7 tok/s**，**瓶颈在端点，不在 dsh**。
- 侧边栏用户感知 ~23s ≈ dsh 内部 TTFT 13.4s + 扩展/渲染开销；与直连 25.6s 同量级。
- 降延迟方向：调低推理强度 / 关推理 / 减少请求上下文，见 tech-notes/dsh-performance-options.md（只读调研，未改动）。

## 关联
- tech-notes/dsh-upgrade-0.1.5.md（升级与 NO_PROXY）
- tech-notes/profiles.md（扩展 ID、侧边栏数值）
- tech-notes/known-facts.md（延迟分档）
- tech-notes/dsh-performance-options.md（推理/标题生成配置可选项）
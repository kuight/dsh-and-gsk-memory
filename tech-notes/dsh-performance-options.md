# 性能相关配置可选项（2026-09-26 只读调研，未改动任何配置）

目的：降低 NVIDIA 端点慢/出字慢的影响。以下均为**可行选项 + 配置位置**，未实施。

## 1. 推理强度（reasoning/thinking）

### 选项 A：settings.yaml 里 provider 级 `reasoning`（llm-pi-ai）
- 位置：`C:\Users\Administrator\.dsh\settings.yaml` → `llm-pi-ai.providers.<provider-id>.reasoning`
- 取值：`z.union(THINKING_LEVELS)`——级别见 `dsh-llm-pi-ai` 的 THINKING_LEVELS（含 off/low/high/max 之类；`thinking_token_budget` / `thinking_budget` / `string-thinking` 等字段也在 schema 中）。
- 作用：控制该 provider 全部模型的推理深度。若端点返回慢，关掉/调低推理可显著减少思考 token 与 TTFT。
- 注意：这是 provider 级；对特定模型可用 `modelOverrides`（`llm-pi-ai.providers.<id>.modelOverrides.<model-id>`）覆盖。

### 选项 B：agent-default-model 的 `reasoningEffort`
- 位置：`settings.yaml` → `agent-default-model.reasoningEffort`（可选字符串）
- 作用：设置未来 Agent 会话的默认推理强度；已有会话可经 `session/selectModel` 的 `reasoningEffort` 逐会话覆盖。

### 选项 C：会话级 `session/selectModel`
- RPC 端点：`POST /api/session/selectModel`，payload 含 `provider` / `model` / `reasoningEffort`（可选）。
- 作用：不写配置文件，当前会话即时切换推理强度。

## 2. 会话标题生成

### 现状
- 首轮标题由 `@deepseek-ai/dsh-session-title-first-prompt-llm` 在**第一轮 prompt 同时**发出 LLM 请求（标题生成 + 主请求并行），会与主请求争用模型端点。
- 标题生成是"first-prompt"模式：仅首轮触发。

### 选项 A：禁用首轮 LLM 标题
- 位置：`C:\Users\Administrator\.dsh\profiles\<profile>\cordis.patch.yml`，entry id = `session-title-llm`（在 dsh-base 的 bundle patch 中注册，name `@deepseek-ai/dsh-session-title-first-prompt-llm`）。
- 写法：`- id: session-title-llm` + `disabled: true`。
- 副作用：会话标题退化为 fallback（截取首条消息前若干字），不再调模型。风险低。

### 选项 B：把标题请求改成首轮之后/单独触发
- 无现成配置项（该插件只有 targetWords/targetCjkCharacters/maxInputBytes/maxOutputTokens/timeoutMs/provider/model）。
- 需要改插件源码或在 patch 里替换该 entry 为自定义实现——成本高，不建议仅为此做。
- 实际观察：标题请求与主请求**同时**发出且同端点，即便只关标题，主请求自身的 TTFT 改善也有限（瓶颈在端点，见 performance-2026-09-26.md 修正结论）。

## 建议（待用户决定）
- 优先试选项 A（provider 级 `reasoning: off` 或低档）+ 观察两次直连波动的量级；标题生成若仍嫌抢带宽可再禁用。
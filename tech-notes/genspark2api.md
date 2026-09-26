# genspark2api 路线：已终止（源码审查证据）

## 结论
- **状态：已终止。** 2026-09-24 前后，本地部署与"驱动 agent"两个目标均被源码级证据否定；用户 2026-09-26 再次确认终止。
- 证据来源：**agent 源码审查**，对象为上游 `genspark2api` commit `99b038b` / v1.12.6（2026-02-27，即当时上游 main 最新提交），clone 位于 `E:\work\genspark2api-src`；部分行号引自 2026-09-23/24 审查记录（`E:\work\PROJECT_MEMORY.md`）。断言行号均来自同一份事实记录，未新增推断。

## 决定性证据（源码级，文件:行号）
1. **不实现 OpenAI 工具调用协议**：
   - `model/openai.go:5-14` 请求体仅 Model/Stream/Messages/ChannelId，无 tools/ToolChoice → 客户端工具参数被静默丢弃
   - `model/openai.go:118-121, 129-132` 消息与流式 delta 均无 tool_calls 字段
   - `controller/chat.go:1381-1398` finish_reason 硬编码 `"stop"`，永不为 `"tool_calls"`
   - 全仓库 grep `tool_calls|ToolCalls|function_call|ToolChoice|parallel_tool` = 0 命中
   - → 该 API 只能当纯文本聊天后端，"驱动 agent"目标不成立
2. **与网页版已是两代契约**：
   - 请求路径：代码 `POST /api/copilot/ask`（`controller/chat.go:29`，流式 `chat.go:1132`），网页版为 ask_proxy
   - type 枚举：`ai_chat`（旧）vs 网页版新枚举（`chat.go:32`）
   - 模型字段位置：代码塞 `extra_data.models` 数组（`chat.go:364,381`）；网页版为顶层标量
   - → 上游最新版本身已落后，非本机取到旧版
3. **静默降级**：不在 `common.TextModelList` 的模型名不报错，替换成 MoA 三模型（`constants.go:80-84`），响应的 model 字段回显请求名（`chat.go:1385`）→ 错误伪装成成功
4. **合规风险**：`chat.go:934/939/946/1282/1287/1294/1539/1559` 的 Warnf 打印完整 cookie，不受 DEBUG 控制 → 违反"凭据不入日志"
5. **Recaptcha**：README 顶部警告不配 RECAPTCHA_PROXY_URL 会降智（`chat.go:985-988` cheat() 原样返回、服务照起但不发 token）
6. **模型可达性**：上游 issue #45（将新版模型描述为"降智"）佐证新模型走旧契约不可靠；目标 claude-opus-5-5 属未被支持的 id（仅能命名推断，无法核实）

## 本机部署障碍（实测观察，非源码）
- 无 Docker / WSL 无发行版 → 免 Docker 方案需 Go ≥1.23.0，本机未装；且裸跑会绑 0.0.0.0（`main.go:55`），违反"只绑 127.0.0.1"约束。

## 勿复勘
- ❌ 不再评估/重启 genspark2api 集成（含免 Docker 构建、自写 playwright 代理等），除非用户明确解除终止。
- ❌ 不再以该项目评估"新模型可调"。

## 沉淀教训（通用）
- 第三方网关判断：字段位置比字段名重要；先看上游 issue 区；"最新版"≠"跟得上"；枚举值差异按移植计量；给方案带失效边界。
- 接管第三方项目前 grep 凭据的日志落点；评估去容器化时容器原有的绑定/隔离语义要回源码层确认。
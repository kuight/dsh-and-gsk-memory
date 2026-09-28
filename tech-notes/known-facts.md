# 已验证事实与勿复勘结论（2026-09-26 汇总）

本节收录已用实测证据确认的结论，防止后续重复排查。每条附证据出处。

## 已验证事实

### 1. NO_PROXY 缺省会让模型请求慢/挂
- 现象：browser profile 曾以裸 node 命令启动（未设 NO_PROXY），扩展发"你好"后 turn/start → 首个 token 历时 **349.3s**，请求组装仅 ~0.6s，无重试（llmRetry 为空）。
- 对照（node 直连 NVIDIA API，key 走 credentials 组件、长度 70 未打印）：
  - 直连（无代理）：首 token 11.2s，完成 11.5s
  - undici 经 127.0.0.1:7897 代理：2ms 内 fetch failed
- 修复：启动脚本添加 `NO_PROXY=...,integrate.api.nvidia.com`（保留 HTTPS_PROXY 其余不变）后恢复。
- 结论：模型出站请求必须豁免代理直连。理由：走代理即慢/挂，直连正常。
- **延迟分档（勿混淆）**：
  - node 直连 NVIDIA（无 dsh、无工具）：首 token **11.2s**（简短回复）与 **25.6s**（300 字回复）——同模型同端点，两次差异显著 → NVIDIA 免费端点延迟波动大，瓶颈在端点不在 dsh
  - browser 侧边栏完整链路（2026-09-26 新用户资料实测）：首 token 约 **23s**；dsh 内部 TTFT 13.4s（本次 ~10.7k 输入 token），用户可见约 23s
  - 最慢路径（无 NO_PROXY）：349.3s（2026-09-25 实测）
- **DEEPSEEK_API_KEY 已失效**：node 直连 api.deepseek.com 返回 HTTP 401 invalid key；该 provider 不可用，计费情况待用户确认。

### 2. Node 22.16 不满足要求、确需升级，但不是卡顿的原因
- monorepo 根 package.json 声明 `engines: node "^22.19.0 || >=24.0.0"`，npm 发布包未带 engines。
- 本机原 F:\Apply\node 为 v22.16.0，低于下限 → 升级到 F:\Apply\node24（v24.21.0）为必要步骤。
- 但升级 Node 后错误现象不变（插件树报错、模型慢均与 Node 版本无关）；卡顿根因实为代理（见上条）。
- 结论：Node 升级是硬性要求，但不是本次问题的解法。

### 3. turn/end `undefined (reading 'length')` 根因 = dsh-memoir（2026-09-27 二分定位）
- 现象：模型正常回复（ASSISTANT 消息落地），但每次 turn 收尾 turn/end 事件带 `reason: {kind:"error", error:{message:"Cannot read properties of undefined (reading 'length')", code:"UNKNOWN"}}`。
- 根因：dsh-memoir `lib/autodistill.js` 注册 `wire.on('agent/turn-stopping', ...)`，调用 `turnActivity(agent.session.events, turn)` 的 `events.length` 未判空；dsh 0.1.5 下 `agent.session.events` 为 undefined → 抛 TypeError。
- 验证链（逐一控制变量）：禁 modlens 仍报 → 禁 memoir 后 turn/end=completed → 恢复 modlens 仍 completed → 根因确证，modlens 系误伤已恢复。
- 同类新发现：dsh-mnemon（`for...of this.agent.session.events` 报 not iterable）、dsh-logicprobe（`session.events.length` 未判空）恢复后分别重现同类错，均回禁等作者适配。
- 排查教训：扫描插件钩子不能只搜 `ctx.on(`——memoir 用的是 `wire.on('agent/turn-stopping')`；且权威 enabled 列表以 `dsh --dump-config` 的 disabled 标志为准（patch 的 id 与 bundle 名不对应会误判）。

### 4. "旧标签页作祟"（来源：dsh 端早期记忆，2026-09-21 已实测解决）
- 原文要点（来源 DSH-HANDOFF.md"二、本次会话已确认的结论"第 4 条）：
  > DSH 界面"一直转圈"的根因（2026-09-21 已解决）：浏览器旧标签卡在已杀进程的死 WebSocket 连接上。服务端本身健康（HTTP/RPC/WS 全测过）。解法=全关 3080 标签页+新开标签。勿再怀疑 DSH 进程死锁。
- 证据：当时对服务端 HTTP/RPC/WS 的实测记录（见 DSH-HANDOFF.md 原文）。
- 结论：web 界面转圈优先怀疑旧标签的死 WebSocket，不要重复排查进程死锁。

## 已证伪（勿复勘）

- ❌ genspark2api 驱动 agent / 本地部署 / 新模型可调——源码级终止，勿复勘（tech-notes/genspark2api.md）。
- ❌ "经 7897 代理也能正常调 NVIDIA"——对照测试 2ms 失败。
- ❌ "modlens-nvidia provider 不存在"——modelCatalog 实测存在且可路由。
- ❌ "升级问题在 Node 版本"——确切说法见"已验证事实 #2"：不满足要求 ≠ 卡顿原因。
### 5. 0.1.7-rc.2 隔离试装（017）新增事实（2026-09-28）

#### 5.1 迁移已执行 —— 【已证伪】

原说法：0.1.7 的 importLegacyDocument 会把手写 settings.yaml 迁进 entry。

实测证据（三项独立一致）：
- profiles\diag-min\settings.yaml（3765 B，18:32 复制）未被改名成 .imported，目录内无 .imported 文件；
- --dump-config 中 agent-default-model 为默认值 provider: deepseek-official / model: deepseek-flash；llm-pi-ai entry 存在但无 config 段；
- rpc-test（端口 3400）返回 AUTH 401，key ****a8a1（已失效的 deepseek-official key）→ 运行时回落默认 provider。

结论：settings.yaml 存在但未被导入。迁移未执行【实测】。

根因：未证实。
- 【复核通过】settings 等 entry 在 dump 中挂着 disabled 条件（dump 第 61-63 行等 5 处，见 dsh-0.1.7-research.md 5.3）；
- 【假设】profileContext 时序导致 entry 禁用——依据只有源文件表达式与 dump 分布，无运行时证据；
- 下会话验证方法：通过设置接口是否存在来判断。

#### 5.2 README 与代码矛盾 —— 以代码为准

- dsh-settings/README.md:35：说 settings.yaml 在 harness home 被导入；
- dsh-settings/lib/index.js:348：join(profile.home, "settings.yaml") —— 实际读 profile.home（= DSH_HOME/profiles/<name>）。
- 以代码为准。相关行号三处：dsh-home-paths/lib/index.js:73-76、dsh-app-boot/lib/index.js:524-527、dsh-app-boot/lib/index.js:485。

#### 5.3 主环境暂不升级 0.1.7-rc.2

- 理由：017 链路至今未跑通（NO_ADAPTER）【实测】。详见 dsh-0.1.7-research.md 5.6。

#### 5.4 日志落盘条件

- 日志只在 StartupError 时落盘。正常启动只写 stderr，不产生 logs/ 文件。
- 后续 a 项检查改为：看 stderr 输出 + cordis.yml 是否被重写（prepareProfile 会重写它）。

#### 5.5 启动方式与端口

- Git Bash 下运行 .cmd 会被拆坏：set DSH_HOME 全部失效。注：cmd /c 要写成 cmd //c —— 未验证。
- 017 验证时用直接 exec 启动，已验证可用。
- 3190 端口的坑：落在 Windows 保留段 3135-3234 内，报 EACCES。改用 3400。
- ~/.dsh 下没有 logs 目录，可作为隔离证据。

#### 5.6 llm-pi-ai 声明与 adapter 注册

- 只补 agent-default-model（provider=nvidia）→ 报错从 401 变为 NO_ADAPTER: no adapter registered for provider "nvidia"。
- 说明：patch 生效了，但 0.1.7 需要在 llm-pi-ai 里声明 provider 才会注册 adapter。

#### 5.7 settings.yaml 结构（只记结构，不记值）

- llm-pi-ai.providers 下有四个段：amd、huggingface、mt、nvidia。
- nvidia 段字段：apiKeyEnv、models。

#### 5.8 安全规则（新增）

复制到 profiles\diag-min\settings.yaml 的整份文件禁止进入仓库（含 1 处疑似 key）。疑似 key 计数 1 处，第 93 行，顶层段名 task-board:，在 llm-pi-ai 段（3-71 行）之外。

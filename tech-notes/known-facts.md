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

### 3. "旧标签页作祟"（来源：dsh 端早期记忆，2026-09-21 已实测解决）
- 原文要点（来源 DSH-HANDOFF.md"二、本次会话已确认的结论"第 4 条）：
  > DSH 界面"一直转圈"的根因（2026-09-21 已解决）：浏览器旧标签卡在已杀进程的死 WebSocket 连接上。服务端本身健康（HTTP/RPC/WS 全测过）。解法=全关 3080 标签页+新开标签。勿再怀疑 DSH 进程死锁。
- 证据：当时对服务端 HTTP/RPC/WS 的实测记录（见 DSH-HANDOFF.md 原文）。
- 结论：web 界面转圈优先怀疑旧标签的死 WebSocket，不要重复排查进程死锁。

## 已证伪（勿复勘）

- ❌ genspark2api 驱动 agent / 本地部署 / 新模型可调——源码级终止，勿复勘（tech-notes/genspark2api.md）。
- ❌ "经 7897 代理也能正常调 NVIDIA"——对照测试 2ms 失败。
- ❌ "modlens-nvidia provider 不存在"——modelCatalog 实测存在且可路由。
- ❌ "升级问题在 Node 版本"——确切说法见"已验证事实 #2"：不满足要求 ≠ 卡顿原因。
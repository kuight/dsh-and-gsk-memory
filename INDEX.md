# 记忆仓库索引（dsh-and-gsk-memory）

## 当前目标
维护 dsh 本机部署的知识：升级、模型链路、插件兼容、dsh-browser 桥与安全，以及已终止路线的"勿复勘"清单。

## 进度快照（2026-09-27）
- ✅ dsh 升级 0.1.1-rc.2 → 0.1.5-rc.3（F:\Apply\node24 + NO_PROXY 修复）
- ✅ diag-min / browser 模型链路验证通过；browser 侧边栏实测首 token 约 23s
- ✅ dsh-browser 装入 browser profile（3081），扩展已连接；bridge 补丁（Origin 精确匹配）已应用并实测
- ✅ **主 profile turn/end undefined.length 根因已定位**：dsh-memoir 的 autodistill `agent/turn-stopping` 监听器访问 `agent.session.events.length` 未判空（0.1.5 下 undefined）；同类新发现 dsh-mnemon、dsh-logicprobe 一并回禁等作者适配
- ✅ **B 类 9 个插件恢复成功**（单句+多步工具任务回归 turn/end completed）：ui-dsh-notifier、dsh-autopilot 4 个 entry、orchestrator、dsh-whale-widget、dsh-worktime-board、genui
- ✅ **A 类 3 个升级恢复**：agent-teams 0.1.13→0.1.21、bridge 0.2.11→0.3.10、imagegen 1.2.2→1.6.5（bundle 数 202/60 无减少）
- ✅ modlens 排除误伤（禁用仍报错、恢复后 completed）
- ⚠️ 最终 17 个 entry 保持禁用（14 个 A 类剩余 + memoir/mnemon/logicprobe）等作者适配
- ✅ approve-for-me 已确认不在 browser profile（hard-rules.md 新增规则：只允许 web）
- ⏳ 记忆仓库本批推送

## 文件用途
| 文件 | 用途 |
|---|---|
| `INDEX.md` | 本文件：索引 + 已验证事实 + 已证伪清单 |
| `tech-notes/dsh-upgrade-0.1.5.md` | 升级全记录：版本/Node/NO_PROXY 坑/对照数据/时间线 |
| `tech-notes/plugin-compat-issues.md` | 28 个禁用 entry 明细（A 确认不兼容 / B 诊断临时禁用） |
| `tech-notes/profiles.md` | 三 profile 用途/端口/启动脚本/通用模式 |
| `tech-notes/bridge-security.md` | 桥 token + Origin 双闸、免 token 前提、已知缺口 |
| `tech-notes/performance-2026-09-26.md` | 侧边栏/直连延迟实测数据 + 结论（瓶颈在端点） |
| `tech-notes/dsh-performance-options.md` | 推理强度/标题生成的配置可选项（只读调研） |
| `tech-notes/genspark2api.md` | genspark2api 终止：源码审查证据链 + 勿复勘 |
| `tech-notes/known-facts.md` | 已验证事实与勿复勘结论（含"旧标签页"来源证据） |
| `tech-notes/hard-rules.md` | 硬规则：禁金融站点、禁 unrestricted browser control |
| `runbooks/start-profiles.md` | 启动各 profile + 验证命令 + 自查清单 |
| `runbooks/upgrade-dsh.md` | 升级流程 + 铁律 + 回滚 |
| `runbooks/memory-repo.md` | 本仓库维护：结构/工具/流程/写记忆准则 |
| `runbooks/bridge-patch.md` | Origin 精确匹配补丁方案（暂缓） |
| `handoffs/2026-09-26-dsh-upgrade-browser.md` | 本次交接摘要 |

## 已验证事实（勿再当疑点重查）
- ✅ **NO_PROXY 缺省会让模型请求慢/挂**：实测走 7897 代理 turn 等待 349.3s 且无重试；node 直连 NVIDIA 正常、同一代理 2ms 失败；启动脚本加 `NO_PROXY=...,integrate.api.nvidia.com` 后恢复。
- ✅ **NVIDIA 免费端点延迟波动大，瓶颈在端点不在 dsh**：node 直连首 token 两次实测 **11.2s 与 25.6s**（同模型同端点），输出约 7 tok/s；browser 侧边栏内部 TTFT 13.4s（用户感知 ~23s）。"预填充导致慢"的旧结论已证伪。证据：tech-notes/performance-2026-09-26.md。
- ✅ **Node 22.16 不满足要求、确需升级，但不是卡顿的原因**：monorepo 声明 `^22.19.0 || >=24.0.0`，22.16 低于下限 → 升级为必要；但换 Node 24 后现象不变，卡顿根因是代理（见上一条）。
- ✅ **"旧标签页作祟"**：来源 DSH-HANDOFF.md（dsh 端早期记忆）——"DSH 界面'一直转圈'的根因（2026-09-21 已解决）：浏览器旧标签卡在已杀进程的死 WebSocket 连接上。服务端本身健康（HTTP/RPC/WS 全测过）。解法=全关 3080 标签页+新开标签。勿再怀疑 DSH 进程死锁。"证据为当时实测记录。详情 tech-notes/known-facts.md。
- ✅ **turn/end `undefined (reading 'length')` 根因 = dsh-memoir 的 autodistill**：`agent/turn-stopping` → `turnActivity(agent.session.events)` 的 `events.length` 未判空；0.1.5 下 `session.events` 为 undefined。禁用后 turn/end=completed（含 modlens 恢复对照组）。同类插件 dsh-mnemon（`for...of session.events` not iterable）、dsh-logicprobe（`session.events.length`）均已回禁。证据：lib 源码行号 + 实测对照，详情 tech-notes/plugin-compat-issues.md。

## 已证伪 / 勿复勘
- ❌ genspark2api 驱动 agent / 本地部署 / 新模型可调——已终止，勿复勘（tech-notes/genspark2api.md）。
- ❌ "经 7897 代理也能正常调 NVIDIA"——对照测试 2ms 失败，勿再按此前提设计。
- ❌ "modlens-nvidia provider 不存在"——modelCatalog 实测存在且可路由，证伪。
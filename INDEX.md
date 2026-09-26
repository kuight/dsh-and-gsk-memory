# 记忆仓库索引（dsh-and-gsk-memory）

## 当前目标
维护 dsh 本机部署的知识：升级、模型链路、插件兼容、dsh-browser 桥与安全，以及已终止路线的"勿复勘"清单。

## 进度快照（2026-09-26）
- ✅ dsh 升级 0.1.1-rc.2 → 0.1.5-rc.3（F:\Apply\node24 + NO_PROXY 修复）
- ✅ diag-min / browser 模型链路验证通过；browser 侧边栏实测首 token 约 23s（见"已验证事实"区分直连/侧边栏）
- ✅ dsh-browser 装入 browser profile（3081），扩展已连接
- ⚠️ 主 profile：28 个 entry 禁用后可加载、可回复，但 turn/end 报 undefined.length（支线未查完）
- ⚠️ provider 由 modlens-nvidia 改为 nvidia 属排障改动；modlens-nvidia 在 NO_PROXY 修复后尚未复测（见 tech-notes/profiles.md）
- ⏳ 记忆仓库内容本批推送
- ⏳ bridge 补丁（Origin 精确匹配）暂缓（用户 2026-09-26 指示暂时不做）

## 文件用途
| 文件 | 用途 |
|---|---|
| `INDEX.md` | 本文件：索引 + 已验证事实 + 已证伪清单 |
| `tech-notes/dsh-upgrade-0.1.5.md` | 升级全记录：版本/Node/NO_PROXY 坑/对照数据/时间线 |
| `tech-notes/plugin-compat-issues.md` | 28 个禁用 entry 明细（A 确认不兼容 / B 诊断临时禁用） |
| `tech-notes/profiles.md` | 三 profile 用途/端口/启动脚本/通用模式 |
| `tech-notes/bridge-security.md` | 桥 token + Origin 双闸、免 token 前提、已知缺口 |
| `tech-notes/genspark2api.md` | genspark2api 终止：源码审查证据链 + 勿复勘 |
| `tech-notes/known-facts.md` | 已验证事实与勿复勘结论（含"旧标签页"来源证据） |
| `tech-notes/hard-rules.md` | 硬规则：禁金融站点、禁 unrestricted browser control |
| `runbooks/start-profiles.md` | 启动各 profile + 验证命令 + 自查清单 |
| `runbooks/upgrade-dsh.md` | 升级流程 + 铁律 + 回滚 |
| `runbooks/memory-repo.md` | 本仓库维护：结构/工具/流程/写记忆准则 |
| `runbooks/bridge-patch.md` | Origin 精确匹配补丁方案（暂缓） |
| `handoffs/2026-09-26-dsh-upgrade-browser.md` | 本次交接摘要 |

## 已验证事实（勿再当疑点重查）
- ✅ **NO_PROXY 缺省会让模型请求慢/挂**：实测走 7897 代理 turn 等待 349.3s 且无重试；node 直连 NVIDIA 首 token 11.2s、同一代理 2ms 失败；启动脚本加 `NO_PROXY=...,integrate.api.nvidia.com` 后恢复。区分两个数值：**node 直连约 11s**（对照实验），**browser 侧边栏实测首 token 约 23s**（扩展侧完整链路，浏览器自带上下文 + 工具清单 + 首 token 渲染）。证据：tech-notes/dsh-upgrade-0.1.5.md、handoffs/2026-09-26-dsh-upgrade-browser.md。
- ✅ **Node 22.16 不满足要求、确需升级，但不是卡顿的原因**：monorepo 声明 `^22.19.0 || >=24.0.0`，22.16 低于下限 → 升级为必要；但换 Node 24 后现象不变，卡顿根因是代理（见上一条）。
- ✅ **"旧标签页作祟"**：来源 DSH-HANDOFF.md（dsh 端早期记忆）——"DSH 界面'一直转圈'的根因（2026-09-21 已解决）：浏览器旧标签卡在已杀进程的死 WebSocket 连接上。服务端本身健康（HTTP/RPC/WS 全测过）。解法=全关 3080 标签页+新开标签。勿再怀疑 DSH 进程死锁。"证据为当时实测记录。详情 tech-notes/known-facts.md。

## 已证伪 / 勿复勘
- ❌ genspark2api 驱动 agent / 本地部署 / 新模型可调——已终止，勿复勘（tech-notes/genspark2api.md）。
- ❌ "经 7897 代理也能正常调 NVIDIA"——对照测试 2ms 失败，勿再按此前提设计。
- ❌ "modlens-nvidia provider 不存在"——modelCatalog 实测存在且可路由，证伪。
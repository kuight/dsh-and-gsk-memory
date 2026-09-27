# Handoff：2026-09-27 dsh-browser 优化 + 主 profile 3080 全量恢复

## 会话目标
1. browser profile（3081）小优化：禁用 session-title-llm、reasoning low/off 会话级接口测试
2. 主 profile（3080）全量恢复：定位 turn/end undefined.length 根因、恢复 B 类 11 个、A 类 17 个查新版升级
3. 收尾：更新记忆仓库并推送

## 完成

### browser profile（3081）
- session-title-llm 已禁用（`C:\Users\Administrator\.dsh\profiles\browser\cordis.patch.yml`）
- reasoning low/off 测试：**nvidia 组 deepseek-ai/deepseek-v4.1-flash 未声明 reasoning efforts（null）**→ 会话级 selectModel 请求被拒，无法测试，仅记基线（reasoning=null）。等模型声明后再测。

### 主 profile（3080）根因定位（★核心成果）
- **turn/end `Cannot read properties of undefined (reading 'length')` 根因 = dsh-memoir 的 autodistill**
  - 链路：`agent/turn-stopping` 监听器 → `turnActivity(agent.session.events, turn)` → `events.length` 未判空；dsh 0.1.5 下 `session.events` 为 undefined → 每次 turn 收尾抛 TypeError
  - 证据链（逐一控制变量）：禁 modlens 仍报 → 禁 memoir 后 turn/end=completed → 恢复 modlens 仍 completed → **根因确认，modlens 系误伤已恢复**
- **同类新发现（均回禁等作者适配）**：
  - dsh-mnemon：`for (const event of this.agent.session.events)` 恢复后报 `not iterable`
  - dsh-logicprobe：`session.events.length` 未判空，恢复后重现 undefined.length
- 教训：之前扫描"13 个顶层第三方 lib 层无 ctx.on"结论有误——**漏了 `agent/` 前缀钩子（wire.on 非 ctx.on）与非 lib 目录文件**；权威 enabled 列表须以 `dsh --dump-config` 的 disabled 标志为准，不能按 patch 的 id 反推

### B 类 11 个恢复结果（单句 + 沙盒建/读/删多步任务，turn/end 均 completed）
- ✅ 恢复成功 9 个：ui-dsh-notifier、dsh-autopilot-service、dsh-autopilot-commands、dsh-autopilot-tools、dsh-autopilot-skills、orchestrator、dsh-whale-widget、dsh-worktime-board、genui
- ❌ 恢复失败 3 个（已回禁，等作者适配）：memoir、mnemon、logicprobe（均为 session.events 兼容问题）

### A 类 17 个版本调研与升级
- ✅ 升级恢复 3 个（bundle 数 202 entry/60 段无减少）：
  - **@nanmicoder/dsh-agent-teams 0.1.13→0.1.21**（peerDeps 明确含 0.1.5-rc.3，作者已适配）
  - **dsh-plugin-bridge 0.2.11→0.3.10**（重构为 inject commands，不再注入退役的 apiProxy）
  - **@dickpy/dsh-imagegen 1.2.2→1.6.5**（无 dsh peerDeps 限制，2026-09-25 新版）
- ⏳ 等作者适配 14 个：approve-for-me（0.2.4 peerDeps 仍指向 0.1.2-alpha 系）、dsh-at-file（0.6.3 无新版）、全部 @linxin666 家族 11 个（最新 0.4.x 要求 `@deepseek-ai/dsh >=0.1.7-rc.1/2`，比 0.1.5 新）、better-sidebar（0.21.1 要求 ^0.1.7-rc.1）

### 其他
- approve-for-me 确认不在 browser profile；hard-rules.md 新增"只允许 web profile"规则
- 新增 E:\work\dsh-run\rpc-test.js：RPC 测试脚本（cookie 鉴权 + session/create + prompt + 轮询 turn/end，退出码 0=completed）
- 测试脚本鉴权要点：先 GET 根路径带上启动日志打印的 query 密钥换签名 cookie（dsh-auth-*），再带 cookie 调 `/api/session/*`；body 需 `rpcId` 字段；prompt 参数为 `{request:{requestId, sessionId, mode:'queue', content:[{type:'text',text}]}}`；page 参数 `throughSeq` 是**包含性**上限

## 待办
- dsh-memoir / dsh-mnemon / dsh-logicprobe 作者适配后恢复（session.events 结构变更）
- @linxin666 家族 / approve-for-me / better-sidebar 等 dsh ≥0.1.7 后升级恢复
- reasoning low/off 等模型声明 efforts 后重测

## 环境要点
- 3080 当前运行：imagegen 恢复版（PID 可能已变，用 netstat 查）；启动脚本 E:\work\dsh-run\start-dsh.cmd
- 禁用 patch：C:\Users\Administrator\.dsh\profiles\web\cordis.patch.yml（17 个）
- 备份：E:\work\backup-dsh-0.1.1\（含各 bisect 中间态 patch 与 package.json）
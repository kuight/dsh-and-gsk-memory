# dsh 0.1.5-rc.3 迁移期插件兼容问题（最终状态 2026-09-27）

位置：`C:\Users\Administrator\.dsh\profiles\web\cordis.patch.yml`（仅 disable，不卸载不删除包）
记录原文：`E:\work\dsh-run\disabled-plugins.txt`
恢复结果：`E:\work\dsh-run\bclass-restore-results.txt`

## 最终禁用：17 个 entry（等作者适配）

### A1 missing-export（12 个）
`@deepseek-ai/dsh-settings` 不再导出 `settingsNamespace` / `installSettingsSection`。
- approve-for-me（dsh-approve-for-me，npm 最新 0.2.4 peerDeps 仍指向 0.1.2-alpha 系，未适配）
- dsh-at-file（0.6.3 无新版）
- web-ui-better-sidebar（dsh-better-sidebar 内嵌；最新 0.21.1 要求 dsh ^0.1.7-rc.1，不满足）
- web-ui-describe-image（@linxin666/dsh-tool-describe-image）
- web-ui-desktop-launcher（@linxin666/dsh-desktop-launcher）
- web-ui-doctor（@linxin666/dsh-doctor）
- web-ui-market（@linxin666/dsh-client-ui-market，最新 0.4.3 要求 dsh >=0.1.7-rc.2）
- web-ui-pet（@linxin666/dsh-pet，要求 >=0.1.7-rc.1）
- web-ui-settings（@linxin666/dsh-client-ui-web-ui-settings，要求 >=0.1.7-rc.2）
- web-ui-skin-center（@linxin666/dsh-client-ui-skin-center，要求 >=0.1.7-rc.1）
- web-ui-ssh（@linxin666/dsh-ssh，要求 >=0.1.7-rc.2）
- web-ui-task-board（@linxin666/dsh-client-ui-task-board，要求 >=0.1.7-rc.2）

### A2 dep-not-found（1 个）
- web-ui-remote-web-ui（@linxin666/dsh-remote-web-ui；@deepseek-ai/dsh-host-apiproxy 0.1.2 退役）

### A3 ctx-service-missing / API 不兼容（4 个）
- better-sidebar（顶层重复注册，settingsNamespace 缺失；与 web-ui 内嵌同包需分别禁）
- bridge（dsh-plugin-bridge）→ **已升级 0.3.10 恢复**（重构为 inject commands，不再注入 apiProxy）
- agent-teams（@nanmicoder/dsh-agent-teams）→ **已升级 0.1.21 恢复**（peerDeps 明确含 0.1.5-rc.3）
- imagegen（@dickpy/dsh-imagegen）→ **已升级 1.6.5 恢复**（无 dsh peerDeps 限制，2026-09-25 新版）

## 新发现根因（2026-09-27 二分定位）：turn/end `Cannot read properties of undefined (reading 'length')`

**根因插件：dsh-memoir**（不在原 28 个禁用列表内，一直启用）。
- `lib/autodistill.js` 注册 `wire.on('agent/turn-stopping', ...)` → `turnActivity(agent.session.events, turn)` → `events.length` 未判空。dsh 0.1.5 下 `agent.session.events` 为 undefined → turn 收尾抛 TypeError。
- 验证链：禁 modlens 仍报 → 禁 memoir 后 turn/end=completed → 恢复 modlens 仍 completed → 根因确认，modlens 系误伤已恢复。
- 状态：memoir 保持禁用，等作者适配。

**同类新发现：dsh-mnemon**（恢复后立即报 `this.agent.session.events is not iterable`）。
- `lib/index.js` openTurn()/7633 行 `for (const event of this.agent.session.events)` 在 0.1.5 下不可迭代。
- 状态：回禁，等作者适配。

**同类新发现：dsh-logicprobe**（恢复后立即重现 undefined.length）。
- `lib/index.js` lastApprovalPolicy()/74 行、planModeActive()/84 行 `session.events.length` 未判空。
- 状态：回禁，等作者适配。

## 恢复成功的插件（2026-09-27 逐个回归：单句 + 沙盒建/读/删文件多步任务，turn/end 均 completed）

B 类 8 个：ui-dsh-notifier、dsh-autopilot-service、dsh-autopilot-commands、dsh-autopilot-tools、dsh-autopilot-skills、orchestrator、dsh-whale-widget、dsh-worktime-board、genui（共 9 个，genui 也归 B）。
A 类升级 3 个：agent-teams 0.1.13→0.1.21、bridge 0.2.11→0.3.10、imagegen 1.2.2→1.6.5。

## 已恢复但需注意
- 多步任务偶发 `PI_AI_ERROR`（Internal server error）：NVIDIA 端点抖动，重测即过，非插件问题。

## 已知重复注册
- better-sidebar：顶层与 web-ui-all 内嵌各注册一次 → 必须分别禁用（`better-sidebar` 与 `web-ui-better-sidebar`）。

## 核对
- 原始 28 = A 类 17 + B 类 11 ✅
- 恢复 12 = B 类 9 + A 类升级 3
- 最终禁用 17 = 原 A 类剩余 14 + 新发现 3（memoir/mnemon/logicprobe）
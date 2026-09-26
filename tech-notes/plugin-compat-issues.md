# dsh 0.1.5-rc.3 迁移期的 28 个禁用 entry 明细

位置：`C:\Users\Administrator\.dsh\profiles\web\cordis.patch.yml`（仅 disable，不卸载不删除包）
记录原文：`E:\work\dsh-run\disabled-plugins.txt`
恢复原则：A 类需等插件修复/替换后再恢复；B 类可逐个恢复并回归测试。

## A 类：确认不兼容（17 个 entry）

原因：0.1.2 大重构引入的 API 变更/退役，报错均已实测确认。

### A1 missing-export（13 个）
`@deepseek-ai/dsh-settings` 不再导出 `settingsNamespace` / `installSettingsSection`。
- approve-for-me（dsh-approve-for-me）
- dsh-at-file
- imagegen（@dickpy/dsh-imagegen）
- web-ui-better-sidebar（dsh-better-sidebar 内嵌）
- web-ui-describe-image（@linxin666/dsh-tool-describe-image）
- web-ui-desktop-launcher（@linxin666/dsh-desktop-launcher）
- web-ui-doctor（@linxin666/dsh-doctor）
- web-ui-market（@linxin666/dsh-client-ui-market）
- web-ui-pet（@linxin666/dsh-pet）
- web-ui-settings（@linxin666/dsh-client-ui-web-ui-settings）
- web-ui-skin-center（@linxin666/dsh-client-ui-skin-center）
- web-ui-ssh（@linxin666/dsh-ssh）
- web-ui-task-board（@linxin666/dsh-client-ui-task-board）

### A2 dep-not-found（1 个）
`@deepseek-ai/dsh-host-apiproxy` 在 0.1.2 ApiProxy 退役时移除。
- web-ui-remote-web-ui（@linxin666/dsh-remote-web-ui）

### A3 ctx-service-missing（3 个）
- better-sidebar（顶层重复注册，settingsNamespace 缺失；与 web-ui 内嵌同包需分别禁）
- agent-teams（@nanmicoder/dsh-agent-teams，subagents.registerContinuableSetup 不存在）
- bridge（dsh-plugin-bridge，inject apiProxy；apiProxy 已退役）

A 类合计：13（A1）+ 1（A2）+ 3（A3）= 17 个 entry。

## B 类：诊断用临时禁用（11 个 entry，未证明不兼容，可逐个恢复测试）

原因：**二分排查** turn/end `undefined.length` 时，将注册了 agent/turn 钩子的插件全部临时禁用；这只是缩小范围的手段，不等于这些插件有错。恢复顺序建议从低频影响者开始，一次恢复一个并回归模型链路。

- mnemon（dsh-mnemon）
- ui-dsh-notifier（@wingsky-1/dsh-notifier）
- dsh-autopilot-service / -commands / -tools / -skills（dsh-autopilot，4 个 entry）
- orchestrator（dsh-orchestrator）
- logicprobe（dsh-logicprobe）
- dsh-whale-widget
- dsh-worktime-board
- genui（@changfenhuang/dsh-genui）

合计核对：A 类 17 + B 类 11 = 28 个 entry ✅

## 已知重复注册
- better-sidebar：顶层与 web-ui-all 内嵌各注册一次 → 必须分别禁用（`better-sidebar` 与 `web-ui-better-sidebar`）。

## 现状
- 主 profile：28 个 entry 禁用后插件树可加载、模型可回复，但 turn/end 仍报 undefined.length（支线未查完，与 B 类禁用范围无关——禁用全部 8 个钩子插件后仍报）。
- B 类说明：恢复任何一个前，先确认 turn/end 支线结论，避免把临时禁用永久化。
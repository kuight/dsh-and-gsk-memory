# 滚动工作日志（新条目追加在末尾）

格式：日期 | 任务 | 结果（标签）| commit

- 2026-09-30 | A4 | 防火墙与手机连接结案，本地提交 b2c9240；直连 github 443 失败未推送 | b2c9240
- 2026-09-30 | 导入 | 桌面端数据目录改名 dsh-desktop.old-0930 后重新弹出导入窗口并导入；多个插件缺 cordis.patch.yml 触发安全模式，最终停用全部第三方插件【用户操作】|
- 2026-09-30 | 会话 | 桌面端 work 下旧会话报 v0-to-v1 source summary requires notice form；browser-sessions 正常【用户截图】|
- 2026-09-30 | 3080 | start-dsh.cmd 能起服务，前端报 Failed to load plugins：dsh-chat-import、dsh-deepread【用户截图】|
- 2026-09-30 | K | ~/.dsh 相对 09-29 备份仅 11 个非插件文件变化；桌面图标 launcher 启动的是 E:\work\dsh-run 的 dsh、3080【实测】|
- 2026-09-30 | K2 | cordis.patch.yml 已备份到 E:\work\cordis.patch.yml.bak-20260930；脚本函数 R 撞内置别名 r，查询段无输出，待重跑【实测】|
- 2026-09-30 | A5 | 新交接、本日志、collaboration 通用约束，随 A4 一起推送 |
- 2026-10-01 | K2b-K2d | web profile 的 cordis.yml 为空清单，插件经 package.json 的 dsh.profile.bundles 加载；包内 id：dsh-deepread=deepread、dsh-chat-import=import-claude（对照 dsh-mnemon=mnemon，与现有禁用写法一致）；两包 08-25 装入【实测】|
- 2026-10-01 | K3 | cordis.patch.yml 末尾追加禁用 deepread、import-claude；改前备份 E:\work\cordis.patch.yml.bak-20261001；3080 重启后效果待用户验证【未实测】|
- 2026-10-01 | 3080 | 禁用 deepread、import-claude 后，前端改报 @linxin666/dsh-client-ui-aionui-panel 缺 @deepseek-ai/dsh-client-runtime/client【用户截图】|
- 2026-10-01 | K4b | dsh-client-runtime 在 web profile 与 E:\work\dsh-run 的 node_modules 中均不存在（dsh-web-app 为 0.1.5-rc.3）【实测】；client 主文件直接引用它的插件 13 个，除 aionui-panel（web-ui-all 内 id 为 web-ui-dsh-aionui-panel）外均已在禁用列表；approve-for-me 包另有 permission 条目未禁用【实测】|
- 2026-10-01 | K5 | cordis.patch.yml 追加禁用 web-ui-dsh-aionui-panel；改前备份 E:\work\cordis.patch.yml.bak-20261001b；3080 重启后效果待用户验证【未实测】|
- 2026-10-01 | 3080 | 禁用 aionui-panel 后，缺模块类报错消失；改报 web boot: 1 entry did not activate，dsh-turn-delete pending (waiting for service: conversationEvents)【用户截图】；该服务由谁提供【未证实】|
- 2026-10-01 | K6 | cordis.patch.yml 追加禁用 turn-delete；改前备份 E:\work\cordis.patch.yml.bak-20261001c；3080 重启后效果待用户验证【未实测】|
- 2026-10-09 | S1 | 状态盘点：3080/3081/3180/3400 均未监听，仅桌面端 43127 在跑【实测】；web cordis.patch.yml 今日 20:01 有改动（deepread/import-claude/aionui-panel/turn-delete 均禁用）【实测】；远端 HEAD=dfefd85 与本地同步【实测】；K6 后插件启用状态【未证实】|

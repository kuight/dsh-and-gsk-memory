# Handoff：2026-09-30 插件与旧会话恢复（最新）

## 1. 入口
- 先读 INDEX.md，再读本文件，再读 handoffs/worklog.md 末尾 20 行；桌面端细节见 tech-notes/desktop-install.md。
- 核对一律用固定 commit 的 raw 地址；执行者汇报不作证据。

## 2. 用户目标
- 恢复旧工程（work 工作区下的会话，搁置已久）及其依赖插件，这是当前最高优先级。

## 3. 当前状态
- 桌面端 v0.10.0（Harness 0.1.7-rc.2）：数据目录已重建并导入网页版数据；原数据目录改名为 %APPDATA%\dsh-desktop.old-0930（内含凭据文件，禁止进仓库）。所有第三方插件已停用【用户操作】。
- 桌面端 browser-sessions 下的会话正常；work 下旧会话报 "session-format-v0-to-v1 refuses this format v0 ... source summary requires notice form"，原始记录未改动【用户截图】。
- 该拒绝是上游有意设计，见 deepseek-harness discussions #6355（v0→v1 边界，截至 09-10 未改）【来源：讨论帖】。dsh-mnemon README 称有 dsh-mnemon-repair-session 可生成修复副本【未实测】。
- 执行者本身运行在桌面端会话里。
- 网页版主环境 3080（0.1.5-rc.3）：E:\work\dsh-run\start-dsh.cmd 能起服务（用户在 cmd 窗口前台运行），浏览器前端报 Failed to load plugins：dsh-chat-import；dsh-deepread 的 loader entry 缺 @deepseek-ai/dsh-client-runtime/client【用户截图】。这两个插件不在 09-27 整理的清单里，来源【未证实】。
- ~/.dsh 相对 09-29 备份只变了 11 个文件（会话缓存、日志、插件数据、两个 cordis.yml），插件目录无近期改动【实测，任务 K】。
- 桌面图标 DeepSeek-Harness 运行 ~\.dsh\desktop-launcher\launcher.ps1（08-22 由 dsh 插件建），启动的是 E:\work\dsh-run 的 dsh、端口 3080，隐藏窗口【实测】。排查时改在 cmd 里运行 start-dsh.cmd，才能看到报错。
- 0.1.5 能否打开 work 下旧会话【未证实】（旧会话可能早于 0.1.5）。

## 4. 下一步
1. 重跑 K2（修正函数名），找出 dsh-deepread、dsh-chat-import 在 cordis.yml 中的条目名；按 cordis.patch.yml 现有写法加入禁用，改之前先给用户看。
2. 重启 3080，打开 "gamebox修复" 会话。
3. 若 0.1.5 也打不开：在副本上试 dsh-mnemon 修复工具。
4. 旧会话能打开后，按工程需要逐个恢复插件，一次一个。

## 5. 备份与回退
- E:\work\backup-dsh-pre-desktop\：~/.dsh，09-29，不含 node_modules 和凭据。
- E:\work\backup-dsh-desktop-pre-import\：导入前的桌面端数据，缺 Network\Cookies 两个文件。
- E:\work\cordis.patch.yml.bak-20260930：web profile 禁用列表。
- 桌面端回退：退出 → 删除新 dsh-desktop → 把 dsh-desktop.old-0930 改回 dsh-desktop → 把其中 harness\.web-import-decision.json.bak 改回原名。

## 6. 协作变化
- 执行者换成桌面端内的新 agent；通用约束、代理端口、记忆推送规则见 runbooks/collaboration.md 末节。
- 用户偏好：一次只给一步；来源不明的配置先问用户，再去查。

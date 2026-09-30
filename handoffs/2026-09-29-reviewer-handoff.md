# Handoff：2026-09-29/30 审核侧交接（DSH Desktop v0.10.0 实装）

## 1. 新对话入口
- 先读 INDEX.md，再读本文件，再读 tech-notes/desktop-install.md；017 相关读 handoffs/2026-09-28-dsh-017-migration-probe.md。
- 核对一律用固定 commit 地址 raw.githubusercontent.com/kuight/dsh-and-gsk-memory/<commit>/<path>；执行者汇报不作证据。
- 写入本文件时的仓库基线 commit：91ca781。

## 2. 当前环境（2026-09-30）
- 主环境 0.1.5-rc.3：3081（browser，PID 19304）在跑【实测】；3080、3180 在 2026-09-29 已不在监听，用户确认不是自己关的，原因【未证实】，按基线对待，未重启。
- 017（0.1.7-rc.2）：3400 / PID 31984 已停【实测】；目录 E:\work\dsh-run-017、E:\work\dsh-home-017 保留。
- DSH Desktop v0.10.0（dataelement）装在 D:\dsh-desktop\DSH Desktop，数据目录 %APPDATA%\dsh-desktop；提供商 nvidia + 手动填模型 ID 可以对话【实测，用户操作】。
- Windows 防火墙已开启；43127 为手机配对桥，入站放行不限来源（用户决定）；手机扫码后要把二维码里的 172.27.0.1 手动换成局域网地址。详见 tech-notes/desktop-install.md 第 6 节。
- DeepSeek 官方 key 已失效；NVIDIA 免费档 40 RPM 由所有 profile 与桌面端共用。

## 3. 本批文件与备份（均在 E:\work）
- 安装前备份 E:\work\backup-dsh-pre-desktop\：排除 node_modules 和凭据；被 *token* 规则误排除的 8 个非凭据文件已由任务 E 补备【实测】。
- 快照 E:\work\snapshot-pre-desktop.txt、E:\work\snapshot-pre-017.txt 保留。
- 安装包 E:\work\installers\manual\dsh-desktop-windows-x64-setup.exe（v0.10.0，sha256 前缀 0449a35c）。
- 禁止进仓库：整个 %APPDATA%\dsh-desktop（含 harness\.credentials.yaml 和带 token 地址的 harness.log）、dsh-home-017\profiles\diag-min\settings.yaml、E:\work\dump-017.txt。
- E:\work 下 probe*、task* 等临时脚本与输出保留未删。

## 4. 未证实项与待办
1. 【已结案 2026-09-30】防火墙已开启，43127 已确认是手机配对桥，手机经局域网可以连上（见 desktop-install.md 第 6 节）。
2. 桌面端模型请求是否走代理：H2 中 443 连接出现在 t=36s，但标题生成警告（北京时间 01:23:10）早于它，该连接不一定是"你好"那次请求【假设】。以后先开采样再发消息。
3. 会话标题生成报 maxOutputTokens 的原因。
4. 继承自 017 交接：apiKeyEnv schema、settings entry 运行时禁用验证、17 个 entry 判定表、3081 浏览器小任务、modlens-nvidia 重测。
5. ~/.dsh\.credentials.yaml.bak（143 B，2026-08-23）来源【未证实】，不读不删。

## 5. 协作经验（本批新增）
- 任务书在传递中被截断过两次（任务 E、F）；现在每份任务书结尾写结束标记，执行者没看到就停止并报告最后一行。
- 长任务拆短；脚本一律先写成 .ps1 文件，再用 powershell -NoProfile 执行，输出用 Out-File 落盘，最后用 Read 按 offset 分页读到 EOF。
- 前台命令别碰执行者约 120 s 的超时；长采样用 Start-Process 后台跑，再轮询 DONE 标记。
- 路径里可能混入零宽字符（任务 H 出现过），复制路径后要复核。
- 审核侧给的预期值也会错（任务 D 的行数、B 的"三个端口都在"）；执行者因不符而停下是正确的。
- 用户确认不等于证据；执行者的推断（如 Chrome 注册表路径、"仓库既有 CRLF 约定"）一律标【未证实】。

- 来源不明的配置（规则、文件、端口）先问用户，用户不知道再去查（防火墙规则那次就是先查后问，白费一轮）。

## 6. 硬约束（沿用，每份任务书都要写）
- 不读取或打印 .credentials.yaml(.bak)、ext-bridge-token 或任何 key/token；报告里不出现带 token 参数的地址。
- 未经用户授权不动 3080/3081/3180、~/.dsh、E:\work\dsh-run；~/.dsh 只读、不递归。
- ~/.dsh 若有变化，先把变更清单交给用户，再重启 profile。

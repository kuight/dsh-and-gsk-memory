# DSH Desktop 实装记录（2026-09-30）

## 1. 版本与来源
- 选定 dataelement/dsh-desktop v0.10.0（不是 v0.10.0-beta）。安装包 dsh-desktop-windows-x64-setup.exe，255264432 B，sha256 前缀 0449a35c，签名 Valid，签名主体 Beijing Shuju Xiangsu Intelligence Technology Co., Ltd.【实测】
- beta 与正式版文件名相同、更新说明相同，但 sha256 不同（beta 前缀 15658991）。下载后必须校验哈希。【实测】
- release 说明称已合入 Harness 0.1.7-rc.2（PR #570），desktop-options.md 第 1、2 节"基于 0.1.7-rc.1"的说法已过时。【来源：release 页】
- 安装目录 D:\dsh-desktop\DSH Desktop；自带 node 位于 resources\app.asar.unpacked\node_modules\node\bin\node.exe【实测】

## 2. 数据目录与隔离
- 数据目录为 %APPDATA%\dsh-desktop，Harness home 为其下的 harness\，profile 名为 web（不是 desktop）。【实测】
- 首次启动会弹出"发现网页版数据"：选择导入会复制会话、设置、密钥和工作区清单，并重装 28 个社区插件（包括 dsh-approve-for-me、dsh-memoir、dsh-mnemon、dsh-logicprobe、@dickpy/dsh-cloud-sync）。本次选的是"从空白开始"。【实测】
- 启动后，用 robocopy /L 把 ~/.dsh 与安装前备份对比，差异为 0；~/.dsh 下三个凭据文件的时间戳没有变化。【实测】
- harness\.credentials.yaml 是桌面端自己的凭据文件：首次启动时 161 B，用户在界面配置 nvidia key 后变为 256 B。只看过元数据。【实测】
- logs\harness.log 会记录带 token 的访问地址。整个 %APPDATA%\dsh-desktop 禁止进仓库。【实测】
- %LOCALAPPDATA%\dsh-desktop-updater\installer.exe 应为更新缓存。【假设】
- 安装前备份：E:\work\backup-dsh-pre-desktop\（不含 node_modules、.credentials*、ext-bridge-token）；安装前快照：E:\work\snapshot-pre-desktop.txt。【实测】

## 3. 网络
- Harness node 监听 127.0.0.1:43129；主进程监听 0.0.0.0:43127。【实测】43127 的用途以及是否有鉴权【未证实】。
- 启动时没有弹出防火墙提示，也没有 dsh 相关的防火墙规则。用户的防火墙保持关闭，网卡为 Public，另外装有 Radmin VPN，因此 43127 可能被局域网或 Radmin 虚拟网访问。【风险，未做连通性测试】缓解办法：不用时从托盘彻底退出；不开手机配对；不需要时断开 Radmin。
- 系统代理 ProxyEnable=1，ProxyServer=127.0.0.1:7897；ProxyOverride 共 21 项，不含 nvidia 和 deepseek；User 级和 Machine 级环境变量都没有设置 HTTP_PROXY、HTTPS_PROXY、NO_PROXY。【实测】
- 从图标启动后，Electron 子进程有连向 7897 的连接。【实测】
- 发送模型请求期间，Harness node 直连外部地址 99.83.136.103:443，没有连 7897。【实测】该 IP 是否为 NVIDIA 端点【未证实】。

## 4. 功能验证
- 设置页、内置插件页（会话插件 29 个，全局插件 191 个）、插件市场页都能打开。【实测】桌面端没有出现 017 的 profileContext 禁用现象，原因【未证实】；"dump 中 name === 'desktop' 判断"这个解释与实际 profile 名 web 不符，不再采用。
- 模型页的提供商下拉框内置 nvidia；模型列表里没有 deepseek-v4.1-flash，手动输入模型 ID 后可以正常对话。【实测，用户操作】
- 会话标题自动生成失败，报错为 session-title-llm: title output reached maxOutputTokens。【实测】原因【未证实】。
- 插件市场 dsh-market 没有安装（社区维护，桌面端提示不做审核）。

## 5. 其他基线记录
- 017 的 3400（PID 31984）已停止。【实测】
- 2026-09-29 时 3080 和 3180 已不在监听，用户确认不是自己关的，原因【未证实】；3081 仍由 PID 19304 运行。【实测】
- ~/.dsh\.credentials.yaml.bak（143 B，2026-08-23）不是用户手动生成的，来源【假设】为首次配置 key 时生成。

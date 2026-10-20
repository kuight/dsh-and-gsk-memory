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
- 2026-10-09 | 插件 | 桌面端安装 web search 插件 @deepseek-ai/dsh-web-search-deepseek（需 key，baseURL 127.0.0.1:43110），改动 ~/.dsh\profiles\web\cordis.patch.yml（20:01）【用户操作，agent 执行】|
- 2026-10-09 | S2 | 旧会话定位：%APPDATA%\dsh-desktop\harness\storages\session_projcache\sessions\ 共 351 文件 2.59MB（含 gamebox 的 1 个 session-7e25b91d）；桌面端配置来源：%APPDATA%\dsh-desktop\harness\profiles\web\（cordis.patch.yml 109KB 23:47 改动）【实测】|
- 2026-10-10 | 归档 | storages 目录 366 文件 3.70MB 已复制到 E:\work\sessions-archive-20261010\，源与目标文件数/字节数一致【实测】|
- 2026-10-12 | 皮肤 | 皮肤插件 @linxin666/dsh-client-ui-skin-center 0.3.6→0.4.6 修复；0.3.6 import 旧 API（installSettingsSection/settingsNamespace）宿主已移除，0.4.6 适配新 API【实测】；备份 cordis.patch.yml.bak-full-20261012-1530|
- 2026-10-13 | 恢复 | 批量恢复 web-ui 家族插件；web-ui-all bundle 带 dsh.migrate 被移进 generations 需绕过 bundle 手动 insert【实测】；备份 bak-webui-batch-20261013|
- 2026-10-13 | 清理 | 删除 09-25 遗留禁用块 41 行 + Bisect 块 18 行，全为 not-found 悬空引用【实测】；备份 bak-before-clean-2331|
- 2026-10-13 | pet | 宠物数据迁移：0.3.6 存 ~/.dsh/pet.json，0.4.6 换 DSH_HOME 存 harness\pet.json；挂载 id 误写 ui-pet 致消失，回滚 ui-pet 恢复【实测】|
- 2026-10-14 | 根因 | GUI 打不开根因=代理环境变量（HTTP_PROXY=127.0.0.1:7897 + NODE_USE_ENV_PROXY=1）劫持 desktop 探活；关全局代理只开规则代理即恢复【用户确认】|
- 2026-10-14 | 卡顿 | worktime-board 每 5 分钟全量 listSessions() 扫描 393MB/356 会话是卡顿元凶；禁用后 session/list 耗时快约 10 倍（4.7s→0.35s）【实测】；用户接受保留禁用|
- 2026-10-14 | 413 | 发图 413 实锤 AMD 网关 5MB 请求体硬限制；llm-pi-ai 自定义 adapter（amd/mt/caw/keyan/nvidia）未实现 imageRequestPricing→图片不压缩内联累积超 5MB【实测】|
- 2026-10-17 | 413 | 大会话 413 死循环确认：371 轮会话解压 38.6MB（1M token 合法但字节超 5MB）；DSH 压缩请求也超 5MB→压缩失败→历史永压不小→每轮 413【实测】|
- 2026-10-18 | 模型 | provider 实测：nvidia 断连（TimeoutError，网络层非 key）、mt 没额度、caw 免费点数不足、AMD 能用但有 5MB 限制【实测】|
- 2026-10-18 | 提醒 | 装 dsh-task-notify 0.4.1（零 runtime 依赖、CJK 安全、适配 0.2.0-rc.2）替代不可用的 dsh-notifier（依赖 MISS 的 dsh-client-runtime）【已装需重启生效】|
- 2026-10-18 | 修复 | 7 个 @linxin666 插件被 pnpm 降级到 0.3.6 import 旧 API 致启动失败；逐一解包验证 0.4.7 兼容（不 import 旧 API、不依赖缺失的 dsh-host-apiproxy），升级 0.4.7 修复【实测】|
- 2026-10-18 | bundle | dsh-our-free-model "cannot resolve profile bundle" 结案：提取 asar dsh-app-boot 的 resolveBundleDir/packageDirFromAnchor 复刻验证可解析，根因=过时报错（node_modules 装好前的旧启动）；重启桌面端挂载成功（cordis.yml 1754-1759 行）【实测】|
- 2026-10-19 | 413根治 | AMD 发图 413 修复：根因=pi-ai 的 maxRequestImageBytes 总预算默认 20MiB 与 AMD 网关 5MB 硬限制不匹配（压缩在跑但停止条件形同虚设）。在 cordis.patch.yml 的 amd provider 块加 maxRequestImageBytes:4194304（4MiB）→ 超预算触发 IMAGE_OFFLOAD_REQUIRED → compaction-image-offload 把旧图转文字 → 不再 413【实测，用户确认修复成功】；备份 cordis.patch.yml.bak-before-amd-imgbudget-20261019192832；副作用=AMD 多发图时最早图转文字引用（图不丢，模型不看原图）|
- 2026-10-19 | 回滚 | maxRequestImageBytes 4MiB 改动引发 AMD 工具调用故障（四种工具拒绝必需参数），回滚该配置恢复；备份保留 bak-before-amd-imgbudget-20261019192832。回滚后 AMD 工具恢复待用户确认【实测】|
- 2026-10-20 | 诊断 | nvidia GLM 大会话 Internal server error 根因确诊：nvidia 网关请求体 25MiB 硬上限（实测 payload below 26214400 bytes）；AMD 5MB 上限。大会话 6a32fd30 解压 52.8MB 超限【实测】|
- 2026-10-20 | 诊断 | compaction 图片 token 错配确诊（核心根因）：compaction 按 token 触发（contextWindow×0.8），但 tokenMeter 对图片调 llm.imageRequestPricing 定价；DS 官方实现→图片按真实视觉 token 算；pi-ai（AMD/nvidia）未实现→回落按"引用 JSON"算（每张~几十token）→图片字节低估→compaction 永不触发→字节爆炸→撞 provider 上限。registerAdapter 覆盖已确认不可行（DUPLICATE_ADAPTER）。详见 tech-notes/compaction-image-token-bug.md【源码+实测确认】|

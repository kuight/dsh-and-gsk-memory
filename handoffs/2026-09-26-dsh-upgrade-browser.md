# Handoff：2026-09-26 dsh 升级 + dsh-browser 安装

## 会话目标
外部升级 dsh（0.1.1-rc.2 → 0.1.5-rc.3）、修复模型链路、安装 dsh-browser、初始化记忆仓库。

## 完成
1. dsh 升级至 0.1.5-rc.3（npm 精确版本，F:\Apply\node24 便携 Node）。
2. 模型链路修复：根因是 HTTPS_PROXY=7897 代理卡 NVIDIA API；启动脚本加 `NO_PROXY=...,integrate.api.nvidia.com` 后直连 ~11s（对照：走代理前 turn 等待 349s）。diag-min / browser 均验证回复「链路OK」。
3. 主 profile：28 个第三方 loader entry 禁用（patch 方式，未卸载），详情 tech-notes/plugin-compat-issues.md。
4. dsh-browser 装进独立 browser profile（3081）——已回滚一次误装到 web 的污染（原版脚本路径写错 web），现 web 干净；扩展已加载，侧边栏已连接。
5. 记忆仓库初始化：git 仓库、.gitignore、gitleaks 8.30.1 + pre-commit hook（含自定义敏感词扫描）已配置并验证。

## 未完成（支线）
- ⚠️ 主 profile turn/end 仍报 `Cannot read properties of undefined (reading 'length')`：8 个钩子插件全禁后模型可回复，但 turn 收尾仍报错——来源未定位（不在已禁 8 插件内）。
- ⏳ bridge 补丁（Origin 精确匹配扩展 ID）：**源码已改、diff 已给用户等待确认**（server.ts `origin === 'chrome-extension://icikjojcmpmokjiojnogdhkocammlepc'`）；确认后 pnpm build + start-browser.cmd 重启；升级 dsh-browser 后需重新应用。

## 2026-09-26 补充（侧边栏实测 + 性能数据 + 补丁已应用）
- 用户在新 Chrome 用户资料加载扩展，侧边栏"你好"首 token 感知 ~23s。
- 实测数据（会话 session-3ce3499a…）：输入 10699 tokens、内部 ttft 13.4s、输出 178 tokens、7.4 tok/s、总 37.4s、无重试；标题请求与主请求同时发出。
- node 直连对照（两次）：11.2s（简短）/ 25.6s（300 字）→ **结论修正：NVIDIA 端点波动大，瓶颈在端点不在 dsh**（"预填充导致慢"已证伪）。DEEPSEEK_API_KEY 401 **失效**，计费待确认。
- 扩展 ID `icikjojcmpmokjiojnogdhkocammlepc` **换用户资料不变**；旧用户资料里的 dsh 扩展已卸载。
- **bridge 补丁已构建并重启**（server.ts 精确匹配扩展 ID）：待用户在侧边栏实测免 token 连接。
- 只读调研（未改动）：推理强度/标题生成的配置可选项 → tech-notes/dsh-performance-options.md。
- 详：tech-notes/performance-2026-09-26.md。

## 待办（下次优先）
1. 查主 profile turn/end undefined.length 支线（扩大 grep 范围找未判空 .length 的监听器）。
2. 若无继续意图，恢复主 profile 被禁插件（分批验证）。
3. bridge 补丁落地。
4. 把本次记忆推送到 GitHub（当前未推送，等用户确认内容后推送）。

## 环境要点
- 进程管理：3080=web、3180=diag-min、3081=browser；启动脚本都在 E:\work\dsh-run\。
- 禁止：读 .credentials.yaml、跑 dsh plugin install/reconcile、删插件数据目录、访问金融类网站、开 unrestricted browser control。
- 备份：E:\work\backup-dsh-0.1.1\（配置齐全）。
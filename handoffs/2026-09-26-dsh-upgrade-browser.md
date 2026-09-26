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
- ⏳ bridge 补丁（Origin 精确匹配扩展 ID）：runbooks/bridge-patch.md 已写方案，**待用户确认后执行**（diff → 重建 → 重启 → 记录）。

## 待办（下次优先）
1. 查主 profile turn/end undefined.length 支线（扩大 grep 范围找未判空 .length 的监听器）。
2. 若无继续意图，恢复主 profile 被禁插件（分批验证）。
3. bridge 补丁落地。
4. 把本次记忆推送到 GitHub（当前未推送，等用户确认内容后推送）。

## 环境要点
- 进程管理：3080=web、3180=diag-min、3081=browser；启动脚本都在 E:\work\dsh-run\。
- 禁止：读 .credentials.yaml、跑 dsh plugin install/reconcile、删插件数据目录、访问金融类网站、开 unrestricted browser control。
- 备份：E:\work\backup-dsh-0.1.1\（配置齐全）。
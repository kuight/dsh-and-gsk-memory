# dsh 升级记录：0.1.1-rc.2 → 0.1.5-rc.3

日期：2026-09-26（含 9-25 夜至 9-26 的多轮排查）

## 结论摘要（每条均附原因）

1. **多 profile 并行是本机 dsh 的管理方式**
   原因：web（主）、diag-min（最小诊断）、browser（dsh-browser 桥）三个 profile 互不干扰，排查模型链路时用 diag-min 而非动 web。

2. **升级目标版本为 npm dist-tag `latest` = 0.1.5-rc.3**
   原因：`next`(0.1.7-rc.2) 更新但仍在 rc；dsh-browser 最低要求 dsh 0.1.5-rc.2，0.1.5 线满足且为官方推荐稳定线。

3. **Node 必须 >= 22.19 或 >= 24**（monorepo 根声明 `^22.19.0 || >=24.0.0`）
   原因：旧 Node v22.16.0 低于下限；官方未在 npm 包上声明 engines（只在源码仓库根声明）。

4. **HTTPS_PROXY 代理会影响出站模型请求（关键坑）**
   现象：经 127.0.0.1:7897 代理访问 NVIDIA API 失败/挂起；直连正常。
   对照：node 直连 NVIDIA 首 token ~11.2s；走同一代理 2ms 内 fetch failed。
   时间线证据：turn/start → 首个 assistant token 历时 ~349.3s（一次请求，无重试），请求组装仅 ~0.6s，提示网络层挂起。
   修复：启动脚本设置 `NO_PROXY=...,integrate.api.nvidia.com`（保留 HTTPS_PROXY 其余不变）。
   原因（为什么保留 HTTPS_PROXY）：NO_PROXY 只豁免 NVIDIA 域，其他出站仍走代理，行为最保守。

5. **升级后老第三方插件批量不兼容，共 28 个 loader entry 禁用**（明细见 tech-notes/plugin-compat-issues.md）
   原因：dsh-settings / dsh-host-apiproxy / subagents 等 API 在 0.1.2 大重构中变更或退役，旧插件引用已不存在的导出或服务。

6. **不要运行 `dsh plugin install` / reconcile 类命令**（#5655 风险）
   原因：社区报告其可能悄悄改写/删减 bundle 列表；任何安装统一走 pnpm 或安装器脚本，操作前备份 package.json。

7. **dsh-browser 不能只靠 `dsh plugin` 安装**
   原因：它同时含 dsh 桥插件 + Chrome MV3 扩展两件套，官方提供 install.sh/install.ps1 一键安装器；npm 未发布可作为普通插件装的包。

## 详细步骤与证据

- 备份：`E:\work\backup-dsh-0.1.1\`（package.json、settings.yaml、dsh-home、profile 配置备份等）。
- 安装：新 Node 便携版 `F:\Apply\node24\`（v24.21.0），`npm install @deepseek-ai/dsh@0.1.5-rc.3 --save-exact`（精确版本，拒绝范围版本）。
- 启动脚本：`E:\work\dsh-run\start-dsh.cmd`（web）、`start-browser.cmd`（browser）、`start-diag-min.cmd`（diag-min）——统一模式：PATH 前置 Node24 + NO_PROXY 豁免 nvidia + `--no-open`。
- 模型链路修复路径：初装即遇插件树加载失败 → 临时裁剪第三方插件 → 最小 profile 仍慢/挂 → 定位为代理干扰 → NO_PROXY 豁免后最小 profile 回复「链路OK」（turn/end completed）。
- 主 profile 现状：模型已能回复（链路OK）但 turn/end 仍报 `Cannot read properties of undefined (reading 'length')`（支线，未查完）。

## 已证伪 / 勿复勘

- ❌ “升级问题是 Node 版本导致” —— 换 Node24 后错误不变，证伪。
- ❌ “modlens-nvidia provider 不存在” —— modelCatalog 显示存在且可路由，证伪；慢的原因是网络层。
- ❌ “8080 端口写死”类假设 —— 桥端口随 dsh 启动端口动态下发（/ext/bridge-config 返回实际 wsUrl），证伪。
- ❌ “最小 profile 也该走 junction 环节” —— 与报错无关。
- ⚠️ “第三方插件是模型链路慢的唯一原因” —— 部分证伪：最小 profile（无第三方）同样慢，根因是代理；第三方插件只额外导致组装期崩溃。
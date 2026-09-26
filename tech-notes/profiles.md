# 三个 profile 的用途、端口与启动脚本

## web（主 profile，3080）
- 位置：`C:\Users\Administrator\.dsh\profiles\web`
- 用途：日常主工作 profile；bundle 含大量第三方插件
- 启动：`E:\work\dsh-run\start-dsh.cmd`
- 注意：当前 28 个 entry 被 patch 禁用（可能是临时的，见 plugin-compat-issues.md）；turn/end 报错支线未查完

## diag-min（最小诊断 profile，3180）
- 位置：`C:\Users\Administrator\.dsh\profiles\diag-min`
- 用途：**模型链路诊断基准**——bundle 只有 `@deepseek-ai/dsh-base` + `@deepseek-ai/dsh-web-app`（llm-pi-ai 由 dsh-base 自带），无任何第三方插件；升级排查的第一步永远是先在 diag-min 上验模型链路
- 启动：`E:\work\dsh-run\start-diag-min.cmd`
- 迁移期结论：只要 diag-min 上 NO_PROXY 设置正确，模型回复 ~11s 级

## browser（dsh-browser 专用，3081）
- 位置：`C:\Users\Administrator\.dsh\profiles\browser`
- 用途：跑 `@yuxianglin/dsh-bridge-browser` 桥插件，供 Chrome 扩展连接
- 启动：`E:\work\dsh-run\start-browser.cmd`
- 端口 3081 在扩展自动发现列表 `[3080,3081,3090,14389,43189]` 内 → 扩展零配置自动连接
- 扩展目录：`C:\Users\Administrator\.dsh\browser-extension`（Chrome 加载已解压扩展用）

## provider 变更记录（排障改动，非定论）

- `C:\Users\Administrator\.dsh\settings.yaml` 的 `agent-default-model.provider` 已由 **modlens-nvidia 改为 nvidia**（model 保持 `deepseek-ai/deepseek-v4.1-flash`）。
- 属排障改动：当时 modlens-nvidia 调用报 `Cannot read properties of undefined (reading 'length')`，切 nvidia 后同错 → 判断与 provider 无关，真因是代理。
- ⚠️ **modlens-nvidia 在 NO_PROXY 修复后尚未复测**——不能断言它本身可用或不可用；恢复条件：在 diag-min 上把 provider 切回 modlens-nvidia，带 NO_PROXY 启动，验证 turn/end 是否 completed。
- 相关备份：`E:\work\backup-dsh-0.1.1\settings.yaml.pre-nvidia-fix`；改动记录见 `E:\work\dsh-run\config-changes.txt`。

## 通用启动模式（三个脚本一致）
```bat
set "PATH=F:\Apply\node24;%PATH%"
set "NO_PROXY=%NO_PROXY%,integrate.api.nvidia.com"
"F:\Apply\node24\node.exe" "E:\work\dsh-run\node_modules\@deepseek-ai\dsh\lib\bin.js" [web|--profile diag-min|--profile browser] --no-open
```
原因：Node24 满足 engines；NO_PROXY 豁免 nvidia 域绕开 7897 代理（否则模型请求慢/挂）。
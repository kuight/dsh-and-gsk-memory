# Runbook：dsh 升级流程（0.1.1-rc.2 → 0.1.5-rc.3 实战总结）

## 前置
- 备份：`E:\work\backup-dsh-0.1.1\`（package.json、settings.yaml、守好 dsh-home 会话/记忆目录；备份排除 .credentials.yaml）
- 目标版本选 npm dist-tag：`latest`（0.1.5-rc.3）优于 `next`（0.1.7-rc.2）；dsh-browser 最低 0.1.5-rc.2
- Node 要求：`^22.19.0 || >=24.0.0`（便携版 F:\Apply\node24 满足；旧 F:\Apply\node 的 22.16 不满足）

## 步骤
1. 停旧进程（记下 PID，`taskkill`）。
2. 新 Node 的 npm 精确安装：`npm install @deepseek-ai/dsh@<版本> --save-exact`（E:\work\dsh-run 内）。
3. 写/更新启动脚本（见 start-profiles.md）：PATH 前置 Node24 + `NO_PROXY=...,integrate.api.nvidia.com`。**本次教训：临时用裸 node 命令启动会丢失 NO_PROXY → 模型请求走 7897 代理慢/挂（349s 实测）。**
4. 启动 → 先验插件树加载无错 → 再验模型链路。
   - 插件加载失败且属于"第三方、导出缺失/依赖包不存在/ctx 服务缺失"三类 → 在 cordis.patch.yml 加 `- id: <entryId>` + `disabled: true`（不卸载包）；记录到 disabled-plugins.txt。
   - entry id 查到实际值再写 patch：bundle 的 cordis.patch.yml 里 insert 的 `id` 才是 patch 目标（例：bridge 而非 dsh-plugin-bridge；dsh-autopilot 有 service/commands/tools/skills 四个子 entry 都要禁）。
5. 模型链路验证：建会话 → 发一句话 → 查 turn/end。**方法**：`POST /api/session/create` + `/api/session/prompt` + `/api/session/page`（RPC 格式 `{"type":"client-request","method":"session/create","payload":{"args":{...}}}`）。

## 铁律
- ❌ 不运行 `dsh plugin install` 或 reconcile 类命令（#5655：可能悄悄改写 bundle 列表）。需要装东西走 pnpm 或官方安装器。
- ❌ 不删第三方插件的数据目录（mnemon/memoir 等可能存记忆）。
- ❌ 不读不打印 .credentials.yaml 内容。

## 回滚路径
- 配置：cp 回 `E:\work\backup-dsh-0.1.1\` 下的 settings.yaml / profile package.json / cordis.patch.yml.bak。
- 程序：npm 装回旧精确版本。

## 已知未解（支线）
- 主 profile turn/end 仍报 `Cannot read properties of undefined (reading 'length')`：8 个钩子插件全禁用后模型可回复但 turn 收尾仍报错——错误位置从"组装前"移到"turn 收尾"，来源未定位（未在已禁 8 插件内）。
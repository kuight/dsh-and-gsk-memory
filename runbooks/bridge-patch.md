# Runbook：bridge 安全补丁（Origin 精确匹配扩展 ID）

## 状态
- **已改源码，未构建、未重启（2026-09-26，diff 已给用户，等待确认）**。
- 补丁文件：`E:\work\dsh-run\dsh-browser-src\dsh-browser-main\packages\browser\bridge-browser\src\server.ts`
- **升级 dsh-browser 后需要重新应用**（本地源码覆盖式安装，仓库更新会冲掉改动）。

## 背景
`bridge-browser/src/server.ts` 免 token 判定原用前缀匹配：
```ts
origin.startsWith('chrome-extension://')
```
→ 任何 Chrome 扩展都能借此免 token 连桥（坏扩展可读浏览器内容）。

## 目标补丁（已改）
```ts
origin === 'chrome-extension://icikjojcmpmokjiojnogdhkocammlepc'
```
- 扩展 ID：`icikjojcmpmokjiojnogdhkocammlepc`
- **该 ID 由"加载已解压扩展"的目录路径派生**；换 Chrome 用户资料目录不变（加载同一路径时，2026-09-26 已实测新资料 ID 不变）；**若挪动 browser-extension 目录到不同路径，ID 会变，必须同步改这里**。

## 应用流程（待用户确认 diff 后执行）
1. 重建桥插件：仓库内 `pnpm --filter @yuxianglin/dsh-bridge-browser run build`
2. 重启 browser profile：`E:\work\dsh-run\start-browser.cmd`
3. 记忆仓库与 `E:\work\dsh-run\config-changes.txt` 同步更新

## 验证
- 改前：无 token 普通网页 WebSocket 被拒（4002）。
- 改后：非本扩展 ID 的 chrome-extension:// Origin 也被拒；本扩展正常连接。
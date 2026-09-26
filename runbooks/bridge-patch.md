# Runbook：bridge 安全补丁（Origin 精确匹配扩展 ID）

## 问题
`bridge-browser/src/server.ts` 中免 token 判定用前缀匹配：
```ts
origin.startsWith('chrome-extension://')
```
→ 任何 Chrome 扩展都能借此免 token 连桥（坏扩展可读你浏览器内容）。

## 目标补丁
改为精确匹配本机扩展的真实 ID：
```ts
origin === `chrome-extension://${EXPECTED_EXTENSION_ID}`
```
`EXPECTED_EXTENSION_ID` 从扩展的 manifest.json 读取（`C:\Users\Administrator\.dsh\browser-extension\manifest.json` 的 key 字段）。

## ⚠️ 前置确认
- **扩展 ID 由什么决定？** Chrome 扩展 ID 对"未打包目录加载（Load unpacked）"模式：**ID 由路径派生**（基于绝对路径哈希）——换用户资料目录/换机器会变；同一路径固定。
- 因此：若用户以后用新的 Chrome 用户资料加载同一目录，ID **可能变化**，补丁需同步更新。
- 结论待现场确认（看 manifest 的 key 字段是否存在、与目录路径的关系）。

## 应用流程（待用户确认 diff 后执行）
1. 确认扩展 ID（读取 manifest key 字段）。
2. 改 server.ts → 精确匹配。
3. 给用户看 diff → 确认。
4. 重建桥插件（仓库内 `pnpm --filter @yuxianglin/dsh-bridge-browser run build`）→ 重启 browser profile。
5. 记录到 `E:\work\dsh-run\config-changes.txt` 与本记忆仓库，注明 **"升级 dsh-browser 后需重新应用"**。

## 验证
- 改前：无 token 的普通网页 WebSocket 被拒（4002）。
- 改后：非本扩展 ID 的 chrome-extension:// Origin 也被拒；本扩展正常连接。
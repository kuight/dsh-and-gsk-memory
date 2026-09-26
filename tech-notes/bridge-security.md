# dsh-browser 桥安全模型（2026-09-26 源码审查结论）

代码：`E:\work\dsh-run\dsh-browser-src\dsh-browser-main\packages\browser\bridge-browser\src\`

## 结构
- 桥插件以独立 profile（browser，3081）运行，挂 `/ext/bridge`（WebSocket）与 `/ext/bridge-config`（HTTP JSON）
- 扩展在 Chrome 里以扩展上下文运行，经 WebSocket 连桥

## `/ext/bridge-config`：无鉴权，但无害
- 无 Origin/token 校验，任何能访问 127.0.0.1:3081 的进程都能读到 `{"wsUrl":"ws://127.0.0.1:3081/ext/bridge"}`
- 原因（为什么接受）：只暴露地址字符串，不含任何 secret；回环地址仅本机可达

## `/ext/bridge`（WebSocket）：token + Origin 双闸
关键逻辑（server.ts）：
```ts
const loopbackNoToken = isLoopbackAddress(remoteAddress)
  && typeof origin === 'string'
  && origin.startsWith('chrome-extension://')
if (!loopbackNoToken && !verifyToken(this.deps.token, frame.token)) {
  ws.close(4002, 'bad token'); return
}
```
- 连接后 5s 内必须先发 `hello` 帧带 bearer token（HELLO_TIMEOUT_MS=5000）
- **免 token 唯一路径**：回环地址 + `Origin: chrome-extension://…`
- 为什么安全：普通网页无法伪造 `chrome-extension://` Origin（浏览器只允许扩展上下文携带该头）；Firefox 的 `moz-extension://…` 含每实例 UUID，不作身份边界 → 仍强制 token
- token：启动时生成随机 hex，持久化 `~/.dsh/ext-bridge-token`（0600），常量时间比较（token.ts）

## 已知缺口（供后续评估）
- `Origin` 校验用的 `startsWith('chrome-extension://')` 是**前缀匹配而非精确扩展 ID**——任何 chrome 扩展都能免 token 连桥（如果它愿意）。已立项修补（精确匹配扩展 ID），见 runbooks/bridge-patch.md。

## 硬规则
- 不开启 unrestricted browser control；不操作银行/支付/证券类网站。
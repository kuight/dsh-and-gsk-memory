# tech-note：compaction 图片 token 错配 —— 大会话无法压缩的根治诊断（2026-10-20）

> 重大诊断，含完整证据链。解决"DSH 大会话在有限制 provider 上堆积到 50MB+ 无法工作"的根因定位。

---

## 现象
- DSH 里**部分大会话**（如 `6a32fd30`，解压后 **52.8MB**、21377 事件、**212 图片行**、653 user 消息）在 **nvidia GLM** 上报 `Internal server error`、在 **AMD** 上报 413。
- **DS 官方（deepseek provider）同一会话正常**。
- **新对话正常**，其他 provider 正常。
- "一个会话稳定工作、另一个死活报错"——是**会话大小/图片量**差别，不是单个会话损坏（DS 官方在同一会话正常）。

## 根因（完整证据链，已实测+源码确认）

**核心一句话**：DSH 的 compaction 按 **token** 触发，但图片 token 由 `imageRequestPricing` 定价；**pi-ai（AMD/nvidia 用的 adapter）没实现它** → 图片 token 被按"引用 JSON"低估（每张 ~几十 token）→ compaction 认为会话还小、**永不触发** → 而实际请求字节（图片 base64）爆炸 → 撞 provider 请求体硬上限 → 拒绝。

### 证据 1：compaction 按 token 触发
`dsh-compaction-basic/lib/index.js`：
```js
thresholdTokens = Math.floor(Math.min(contextWindow * policy.thresholdRatio, ...));  // thresholdRatio 默认 0.8
if (measurement.totalTokens < spec.thresholdTokens) return null;  // 不压缩
```
**完全按 token**（contextWindow × 80%），不看字节。

### 证据 2：tokenMeter 对图片调 imageRequestPricing
`dsh-token-meter/lib/index.js`（measure 内）：
```js
return this.ctx.get('llm')?.imageRequestPricing(config.provider, config.model);
```
token-meter 对图片定价时调 `llm.imageRequestPricing`。

### 证据 3：DS 官方实现了、pi-ai 没实现
- `dsh-llm-deepseek/lib/index.js`：**有** `imageRequestPricing`（`deepSeekImageRequestPricing`，返回每张图 `visualTokens`，按真实尺寸/像素算）→ compaction 能正确触发。
- `dsh-llm-pi-ai/lib/index.js`（112KB）：**有** `registerAdapter`、`resolveImageAccess`、`readImageRequest`，但 **无** `imageRequestPricing`（全 asar 搜 pi-ai 相关文件 0 命中）→ 返回 undefined → token-meter 回落用 `estimateContent`。
- nvidia adapter（`@earendil-works/pi-ai/dist/providers/nvidia.js`）：同样无 imageRequestPricing，且**不压缩图片**（`readImageRequest:false`）。

### 证据 4：token-meter 对图片的回落估算 = 引用 JSON
`dsh-token-meter/lib/types/estimate.js`：
```js
estimateStructuralBlock(block) {
  if (block.type === 'image') { const { offloaded, ...reference } = block;
    return BLOCK_OVERHEAD + Math.ceil(JSON.stringify(reference).length / CHARS_PER_TOKEN); }
  ...
}
```
**图片只算"引用 JSON"（attachmentId/mediaType/bytes/width/height ≈ 几十 token）**，不算实际图片字节。

### 证据 5：provider 请求体硬上限（直接实测）
- **nvidia**：`payload is below 26214400 bytes`（25MiB 硬上限）
- **AMD**：`Request body is 5242970 bytes, over the 5242880 byte limit`（5MB 硬上限）
- DS 官方：无此限制（能收大会话请求）

## 错配链条
1. 会话 212 张图 → token 估算每张 ~几十 token（引用 JSON）
2. 发请求时图片引用解析成完整 base64（每张 2-3.5MB）
3. **token 估算远低于 contextWindow 80% → compaction 不触发**
4. **实际请求字节几百 MB → 超 nvidia 25MB / AMD 5MB → 拒绝**
5. 无法压缩（请求发不出去）→ 历史永不减小 → 死循环

## 为什么 DS 官方正常
`dsh-llm-deepseek` 实现了 `imageRequestPricing` → 图片 token 按真实视觉 token 算 → compaction 能在字节爆炸前触发 → 会话能自动压缩 → 不超 provider 上限。

## 为什么 AMD/nvidia 不行
pi-ai（AMD/nvidia）没实现 `imageRequestPricing` → 图片 token 低估 → compaction 永不触发 → 字节爆炸。

## 修复方向
**治本**：让 AMD/nvidia 的 adapter 实现 `imageRequestPricing`，图片 token 按真实尺寸算 → compaction 正确触发。

尝试路径（按优先级）：
- **A 改 asar pi-ai**：最直接但改内置、不可回滚、风险高。
- **B profile override 补丁版 pi-ai**：可回滚，但 pi-ai 有 9 个 peerDeps（0.2.0-rc.2 都在 asar，profile 没有）→ pnpm 装不进 → 复杂。
- **C profile 挂独立小插件 patch `llm.imageRequestPricing`**：给 AMD/nvidia 返回正确图片 token，不动 pi-ai/asar，可回滚，最安全。**待验证 DSH 是否允许插件 patch llm service 方法。**
- **registerAdapter 覆盖**：**已确认不可行**（`dsh-llm` prepareRoutes 对已注册 provider 抛 `DUPLICATE_ADAPTER`）。

## 关联的旧问题（同根）
- **AMD 发图 413**：也是 pi-ai 未实现 imageRequestPricing 致图片不压缩/预算高。改 `maxRequestImageBytes` 20MiB→4MiB 曾缓解 413，但引发工具调用故障，已回滚（备份 bak-before-amd-imgbudget-20261019192832）。
- **nvidia GLM 长会话 Internal server error**：nvidia 25MB 硬上限 + 图片不压缩。

## 结论
DSH compaction 的图片 token 估算是**设计缺陷**：按"引用"算（低估）而非"真实字节/视觉 token"。DS 官方因实现 imageRequestPricing 而幸免，AMD/nvidia 因 pi-ai 未实现而中招。根治需给 pi-ai 补 imageRequestPricing（方案 A/C）。
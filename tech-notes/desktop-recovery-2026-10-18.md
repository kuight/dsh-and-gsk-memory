# tech-note：DSH Desktop 关键结论补录（2026-10-12~10-18）

> 补录 10-11 之后仓库缺的技术结论。事实来源：实测 + 会话历史 `dsh-browser.md`。每条带证据。

---

## 1. AMD 413（Payload Too Large）根因 —— AMD 网关 5MB 请求体硬限制

**结论**：AMD 网关 `developer.amd.com.cn/radeon/api/v1` 请求体上限 **5MB（5242880 字节）**，超限返回 HTTP 413。

**实测证据**（直接测网关，非猜）：
- 10KB / 1MB 请求体 → 401（正常鉴权路径）
- 5MB / 20MB → 413，响应体明确：`Request body is 5242970 bytes, over the 5242880 byte limit`

**机制链（为何发图就 413）**：
1. 发图 → `spill-policy` 调 `imageRequestPricing(provider, model)`
2. `llm-pi-ai` 自定义 adapter（amd/mt/caw/keyan/nvidia）**全未实现 imageRequestPricing**（官方 `dsh-llm-deepseek` 实现了）
3. 抛 "no image token calculator" → 图片不压缩、不 spill，直接 base64 内联
4. 会话图片累积超 5MB → 413

**大会话死循环（10-17 确认）**：
- 371 轮会话解压后 **38.6MB**（1M token 合法，但**字节远超 5MB**——token 限制与字节限制是两码事）
- DSH 想靠 compaction 压缩历史自救，但**压缩请求本身也超 5MB** → 压缩失败 → 历史永不压缩 → 每轮 413
- compaction 失败 24 次中 10 次是 413
- 同一份历史：nvidia 收（无 5MB 限制）正常；AMD 拒（5MB 限制）413

**解法**：
- 大对话/发图用 caw（实测发图成功）、nvidia、mt
- AMD 只适合历史小的新会话
- **offload 机制不救**：`compaction-image-offload` 只处理 DSH 的 `IMAGE_OFFLOAD_REQUIRED` 业务错误，**不处理**网关直接抛的 HTTP 413
- **勿手动挂载内置插件**：`compaction-image-offload` 的 projection `image/offload` **已随 harness 内置注册**，手动 insert 报 `Error: session message projection "image/offload" is already registered` → 启动失败。回滚 `bak-before-offload-20261014` 恢复

### 根治方案（2026-10-19 实测，用户确认修复成功）

**修正认知**：pi-ai **其实会压缩图片**（每张压到 ≤1 MiB，`requestImageMaxBytes` 默认 1MiB + sharp 阶梯重编码），但**总预算 `maxRequestImageBytes` 默认 20 MiB**，远高于 AMD 网关 5MB → 压缩器判定"base64 总量 <20MiB 不丢弃旧图" → ≥5 张图累积超 5MB 但仍 <20MiB → 整包发出 → AMD 413。**不是不压缩，是停止条件设太高形同虚设**。

**关键字段**（`dsh-llm-pi-ai/lib/index.js` 的 provider schema，provider 级）：
```js
DEFAULT_MAX_REQUEST_IMAGE_BYTES  = 20 * 1024 * 1024   // 总预算（问题根源）
DEFAULT_REQUEST_IMAGE_PIXEL_BUDGET = 2048 * 2048       // 每图像素预算
DEFAULT_REQUEST_IMAGE_MAX_BYTES   = 1024 * 1024        // 每图压缩目标
Config = { providers: z.dict(profile) }                 // 字段在 provider profile 级
```

**修复**：在 `cordis.patch.yml` 的 `amd:` provider 块加：
```yaml
amd:
  displayName: amd
  apiKeyEnv: AMD_API_KEY
  api: openai-completions      # 走 pi-ai 的 openai-completions 适配器
  baseURL: https://developer.amd.com.cn/radeon/api/v1
  maxRequestImageBytes: 4194304   # ← 4MiB，低于 AMD 5MB 网关
  models:
```

**机制**：设 4MiB 后，请求里 base64 图总量逼近 4MiB → pi-ai 抛 `IMAGE_OFFLOAD_REQUIRED_CODE` → 已挂载的 `dsh-compaction-image-offload` 把最旧图 offload 成文字引用 → 请求体变小 → 不撞 AMD 5MB。**不再 413**。

**前提**：`dsh-compaction-image-offload` 必须已挂载（cordis.yml 566-567 行 `- id: image-offload`，内置，确认已挂）。若它被禁用，改低预算会从"优雅转文字"变"直接报图片超预算错"。

**副作用**：AMD 上发图超过约 4 张时，**最早的图会转成文字引用**（图不丢，附件仓库仍在，但模型不再看到原图）。只影响 AMD，caw/nvidia/mt 不受影响。

**回滚**：删掉该行重启即恢复默认 20MiB；备份 `cordis.patch.yml.bak-before-amd-imgbudget-20261019192832`。

**为什么 Cherry Studio 不 413**：Cherry 发图前做了压缩/预算设得低，与 DSH 的 AMD 链路（20MiB 预算）不同。

---

## 2. dsh-our-free-model 本地 bundle 解析（resolve 机制 + 过时报错结论）

**核心机制**（提取 asar `@deepseek-ai/dsh-app-boot/lib/index.js`）：
```js
function resolveBundleDir(binName, packageName, installAnchor, profileDir) {
  for (const anchor of [installAnchor, join(profileDir, "package.json")]) {
    const dir = packageDirFromAnchor(anchor, packageName);
    if (dir !== void 0) return dirinston;
  }
  throw new Error(`cannot resolve profile bundle ...`);
}

function packageDirFromAnchor(anchor, packageName) {
  for (const searchPath of createRequire(anchor).resolve.paths(packageName) ?? []) {
    const candidate = join(searchPath, packageName);
    if (existsSync(join(candidate, "package.json"))) return candidate;  // 查候选目录下 package.json 存在
  }
}
// realModuleDirectory = realpathSync.native(path)
```

**关键点**：
- 解析用 `createRequire(anchor).resolve.paths()` 找 node_modules 查找路径，检查 `join(candidate, "package.json")` 存在
- 锚点顺序：先 installAnchor（asar 内 dsh 的 package.json），再 `profileDir/package.json`
- 解析**不要求**包导出 `./package.json`

**"cannot resolve" 结论**：`dsh-our-free-model` 复刻验证**能解析**（profile anchor 命中，manifest 存在，host 半身 `index.js` 导入成功）。报错是**过时的**——node_modules 装好**之前**那次启动产生的。重启桌面端后 `cordis.yml` 1754-1759 行挂载成功。

**正确装本地 bundle**：`package.json` dependencies 加 `"<name>":"file:../../plugins/<name>"` + 包名进 `dsh.profile.bundles` + profiles/web 的 pnpm install。不需要 `dsh plugin install`，不需要动 desktop-host-sources.patch.yml。顺序 = resolve → 执行 bundle 自带 patch → 挂载。

---

## 3. @linxin666 插件 0.4.7 兼容性（旧 API vs 新 API）

**问题**：7 个 @linxin666 插件启动失败 `installSettingsSection`/`settingsNamespace` + 缺 `dsh-host-apiproxy`。

**根因**：被 pnpm 降级到 0.3.6。asar 内置 `dsh-settings` 只导出新 API：
```js
export { SettingsConflictError, SettingsForms, SettingsForms as default, redactSecrets }
```
0.3.6 import 的是已被移除的 `installSettingsSection`/`settingsNamespace`。

**0.4.7 全兼容验证**（逐一解包）：
- 全部不再 import 旧 API `installSettingsSection`/`settingsNamespace`
- `dsh-client-ui-web-ui-settings` 0.4.7 用 `SettingsConflictError`（新 API 导出，OK）
- `dsh-pet` 0.4.7 的 `settingsNamespace` 是**它自己内部服务字段**（settingsNamespaceProvider），与 dsh-settings 包无关
- `dsh-host-apiproxy` **asar 里不存在**，但 0.4.7 的 `dsh-remote-web-ui` 已移除该依赖（改用 `dsh-client-connection`，asar 有）

**peerDeps 版本注意**：0.4.7 要求 `@deepseek-ai/dsh >= 0.2.1-rc.2`。0.4.7 能在 asar 内核跑，说明内核满足。

**安装入口铁律**：一律走 `profiles/web` 的 pnpm（`.desktop-bin/pnpm.cmd`，pnpm 11.22.0，经 pnpm-runner 用 DSH Desktop.exe 的 Electron node 跑）。**别用系统 `dsh.cmd`**——那是 0.1.7-rc.2 的另一环境，与桌面 0.2.0-rc.2 不匹配，`plugin install` 无输出不生效。

---

## 4. GUI 打不开 / 卡顿 / pet 相关

### GUI 打不开根因 = 代理环境变量
`HTTP_PROXY/HTTPS_PROXY/ALL_PROXY=127.0.0.1:7897` + `NODE_USE_ENV_PROXY=1`（Clash Verge）→ Electron fetch 探活走代理被劫持。解法：关全局代理只开规则代理。探活 URL 裸根 `http://127.0.0.1:43129`，判定 `launchToken 已解析 && 200≤status<500`（401 也健康）。启动必须 `explorer.exe`。

### 卡顿根因 = worktime-board 全量扫描
`dsh-worktime-board` 每 5 分钟 `ctx.sessionQuery.listSessions()` 扫 393MB/356 会话。禁用后 session/list 快约 10 倍（4.7s→0.35s）。

### pet 三件套 + 数据迁移
- 修仙值=dsh-worktime-board、钱包=dsh-whale-widget（BALANCE_URL=/dsh-whale/balance.json）、宠物=dsh-pet（**挂载 id 必须 ui-pet**）
- 数据迁移：旧 `~/.dsh/pet.json` → 新 `%APPDATA%\dsh-desktop\harness\pet.json`

---

## 5. asar 包读取技术要点

- `@deepseek-ai/dsh` 等可能是 **unpacked**（offset 为 null）或偏移在别处；叶子节点结构是 `{size, offset}`（**offset 是字符串**），直接 `node.offset`/`node.size` 取，**不是** `node.data.offset`
- `h.files` 就是根目录节点，children 在 `h.files.node_modules.files[...]`
- dsh 核心 lib 是**打包成单文件 chunk**（bin.js/plugin-*/profile-boot.js）
- asar 里 package.json 可能带 BOM 前导（`﻿`），JSON.parse 前先去前导
- 报错串 "cannot resolve profile bundle" 不在 asar JS 里（动态拼接），但在 `out/main/index.js` 有正则匹配用于插件失败分类

---

## 6. provider 连通性实测表

| provider | 端点 | 实测 | 结论 |
|---|---|---|---|
| nvidia | integrate.api.nvidia.com | 根 404、POST /v1/chat TimeoutError、api.nvidia.com 000 | 网络层，非 key |
| mt | - | 429 insufficient balance | 没额度 |
| caw | api.chatanywhere.tech | 网关 200 | 正常，仅免费点数不足 |
| amd | developer.amd.com.cn/radeon/api/v1 | 5MB 内正常 | 能用但有 5MB 限制 |
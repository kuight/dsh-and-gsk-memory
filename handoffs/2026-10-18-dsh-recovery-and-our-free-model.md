# 交接：插件恢复 + @linxin666 0.4.7 修复 + dsh-our-free-model 解析结案 + AMD 413 根因（2026-10-18）

> 本交接补录 2026-10-12 至 2026-10-18 的 DSH Desktop 工作（此前仓库只记到 10-10）。事实来源：会话对话历史 `dsh-browser.md` + 本机实测。**第一入口仍是 `E:\work\DSH-HANDOFF.md`**，本文件为仓库补录，信息与其一致。

---

## 〇、补录范围说明

仓库 `dsh-memory`（= GitHub `kuight/dsh-and-gsk-memory`）此前 worklog 只到 10-10。本次补录 10-12~10-18 的关键工作，覆盖：
1. 批量插件恢复（web-ui 家族 + pet 三件套 + 皮肤）
2. @linxin666 插件 0.3.6→0.4.7 升级修复（启动失败）
3. dsh-our-free-model 本地 bundle 解析结案
4. AMD 413（5MB 限制）根因实锤
5. GUI 打不开根因（代理环境变量）

---

## 一、GUI 打不开根因（2026-10-14 凌晨定位）

- **现象**：harness 进程正常、HTTP 健康，但 desktop 探活永远不过 → 180s SIGTERM。
- **根因**：环境变量 `HTTP_PROXY=http://127.0.0.1:7897`、`HTTPS_PROXY`、`ALL_PROXY`、`NODE_USE_ENV_PROXY=1`（Clash Verge 装）。Electron 主进程 fetch（探活用）走代理被劫持到 7897。
- **解法**：**关掉全局代理、只开规则代理**即可（用户确认）。探活 URL 是裸根 `http://127.0.0.1:43129`（不带 token），判定 `launchToken 已解析 && 200≤status<500`（401 也算健康）。
- **启动方法铁律**：必须 `explorer.exe "D:\dsh-desktop\DSH Desktop\DSH Desktop.exe"`（ShellExecute），直接 Start-Process/命令行会闪退。

---

## 二、批量插件恢复（10-12~10-14）

### 恢复方式
- **皮肤/搜索**：升级 0.4.6（仅 dsh-pet 0.4.6 兼容 0.1.7-rc.2；其余 6 个 0.4.6 要求 dsh≥0.2.0-rc.2 需找 0.4.x 早期版）。
- **web-ui-all bundle 坑**：bundle 带 `dsh.migrate` 标记（`{"to":"@deepseek-ai/dsh-web-all","since":"0.3.6"}`），桌面端把它移进 "generations" 弃用归档 → 皮肤/UI 条目不加载 → **须绕过 bundle，在 cordis.patch.yml 手动 insert**。
- **验证基线**：10-01 成功启动（node v24.9.0 unpacked）；10-09 asar 更新（node v24.17.0 入 asar）。

### 已成功恢复
skin-center（0.4.6）、web-search-thirdparty、web-ui-settings、market、plugin-manager、task-board、remote-web-ui、ssh、pet（0.4.6）、whale-widget、worktime-board、better-sidebar、describe-image、doctor、desktop-launcher、logicprobe、memoir。

### 恢复失败/撤除
- **mnemon**：0.3.6 旧 API 崩（ctx.settings.register 不存在）；0.5.28 需装 20 个 dsh-mnemon-* 子包工作量大，**用户选跳过**。
- **memoir**：client 依赖 `@deepseek-ai/dsh-client-runtime`，宿主 asar 无此包 → 摘除。
- **better-sidebar/describe-image/doctor/desktop-launcher**：renderer 报 `uiConversation.binding: unknown session` → 摘除。
- **approve-for-me/dsh-at-file**：npm 无新版，旧 API，倾向不恢复。

### pet 三件套（2026-10-14 恢复）
- 修仙值/牛马值 = `dsh-worktime-board`（09-25 误禁 → 重挂）
- 钱包/token 消耗 = `dsh-whale-widget`（`/dsh-whale/balance.json`，`BALANCE_URL`、`REFRESH_MS=60000`）
- 小鲸鱼宠物 = `dsh-pet`：**挂载 id 必须 `ui-pet`**（改 `pet` 宠物本体消失）
- pet 0.4.6 client 依赖 `dsh-client-store`（asar 有，OK），有 fallback 到 MISS 的 dsh-client-runtime
- **数据迁移**：旧 `~/.dsh/pet.json` → 新 `%APPDATA%\dsh-desktop\harness\pet.json`

### 卡顿根因（10-14）
`dsh-worktime-board` 每 5 分钟全量 `ctx.sessionQuery.listSessions()` 扫描 393MB/356 会话。**禁用后 session/list 快约 10 倍**（4.7s→0.35s）。用户接受保留禁用。

---

## 三、@linxin666 插件 0.4.7 升级修复（10-18）

- **现象**：7 个 @linxin666 插件启动失败：`installSettingsSection`/`settingsNamespace`（旧 API）+ 缺 `dsh-host-apiproxy`。
- **根因**：被 pnpm 降级到 **0.3.6**（import 旧 API），但 asar 内置 dsh-settings 只导出新 API（`SettingsForms`/`SettingsConflictError`/`redactSecrets`）。
- **验证 0.4.7 全兼容**（逐一解包）：
  - 不再 import `installSettingsSection`/`settingsNamespace`
  - web-ui-settings 用 `SettingsConflictError`（新 API，OK）
  - pet 的 `settingsNamespace` 只是内部字段，无关
  - **dsh-host-apiproxy 在 asar 不存在**，但 0.4.7 remote-web-ui 已移除该依赖（改用 `dsh-client-connection`）
- **解法**：升级到 0.4.7（走 profiles/web 的 pnpm，用 pnpm-runner，别用系统 dsh.cmd——那是 0.1.7-rc.2 另一环境）。

---

## 四、dsh-our-free-model 本地 bundle 解析结案（10-18）

- **现象**：`cannot resolve profile bundle "dsh-our-free-model"`，但 package.json 已声明 + node_modules 有。
- **根因**：**过时报错**。是 node_modules 装好**之前**那次启动产生的，装完后没重启所以一直显示旧错。
- **复刻验证**（提取 asar `dsh-app-boot/lib/index.js`）：
  ```js
  function resolveBundleDir(binName, packageName, installAnchor, profileDir) {
    for (const anchor of [installAnchor, join(profileDir, "package.json")]) {
      const dir = packageDirFromAnchor(anchor, packageName);
      if (dir !== void 0) return dir;   // createRequire(anchor).resolve.paths + 查 package.json
    }
    throw new Error(`cannot resolve profile bundle ...`);
  }
  ```
  `packageDirFromAnchor` 用 `createRequire(anchor).resolve.paths()` 找候选，检查 `join(candidate,"package.json")` 存在；`realModuleDirectory` = `realpathSync.native`。
- **实测**：复刻 `resolveBundleDir('dsh','dsh-our-free-model', installAnchor, profileDir)` 解析成功；host 半身 `index.js` 导入 OK。
- **最终**：重启桌面端，`cordis.yml` 第 1754-1759 行生成挂载记录，harness 健康（43129 返回 401）。

### 正确装本地 bundle 方式（回答求助稿）
1. `package.json` 加 `"<name>": "file:../../plugins/<name>"` 到 dependencies + 包名进 `dsh.profile.bundles`
2. 走 `profiles/web` 的 pnpm install（pnpm-runner）
3. **不需要** `dsh plugin install`，**不需要**动 desktop-host-sources.patch.yml
4. plugin 自带 `cordis.patch.yml` 在 bundle 解析成功后自动合并进 `cordis.yml`
- **顺序**：resolve bundle → 执行 bundle 自带 patch → 挂载（求助稿猜测正确）

---

## 五、AMD 413 根因实锤（10-14~10-17）

- **根因**：AMD 网关 `developer.amd.com.cn/radeon/api/v1` **请求体硬限制 5MB（5242880 字节）**。直接实测：10KB/1MB→401，5MB/20MB→413，响应明确 "over the 5242880 byte limit"。
- **机制链**：发图 → `spill-policy` 调 `imageRequestPricing(provider,model)` → **`llm-pi-ai` adapter（amd/mt/caw/keyan/nvidia）全未实现**（官方 dsh-llm-deepseek 实现了）→ 抛 "no image token calculator" → 图片不压缩内联 → 累积超 5MB → 413。
- **大会话死循环**：371 轮会话解压 38.6MB（1M token 合法但字节超 5MB）。DSH 压缩请求也超 5MB → 压缩失败 → 历史永压不小 → 每轮 413。compaction 失败 24 次中 10 次是 413。
- **解法（已根治 2026-10-19，用户确认）**：根因不是"不压缩"而是"总预算 maxRequestImageBytes 默认 20MiB 与 AMD 5MB 网关不匹配"。在 `cordis.patch.yml` 的 amd provider 块加 `maxRequestImageBytes: 4194304`（4MiB）→ 超预算触发 `IMAGE_OFFLOAD_REQUIRED` → compaction-image-offload 把旧图转文字 → 不再 413。详见 tech-notes/desktop-recovery-2026-10-18.md 根治方案。
- **副作用**：AMD 多发图时最早图转文字引用（图不丢，模型不看原图）。只影响 AMD。
- **回滚**：删该行重启；备份 cordis.patch.yml.bak-before-amd-imgbudget-20261019192832。
- **offload 前置**：`compaction-image-offload` 已内置挂载（cordis.yml 566 行）；projection `image/offload` **已内置注册**，手动 insert 会 `already registered` 报错 → **勿手动挂载内置插件**。

---

## 六、provider 连通性/额度实测（10-18）

| provider | 状态 | 详情 |
|---|---|---|
| nvidia | 断连 | TimeoutError；`integrate.api.nvidia.com` 根 404（主机通）但 POST /v1/chat 超时；`api.nvidia.com` 直连 000。网络层（Clash 代理/直连差异），非 key，无法配置修复 |
| mt | 没额度 | 429 insufficient balance |
| caw | 点数不足 | 网关 200 正常，仅免费额度不够 |
| amd | 能用但 5MB 限制 | 只适历史小的新会话 |

---

## 七、其余

- **dsh-task-notify 0.4.1**：装任务完成提醒，零 runtime 依赖，CJK 安全，适配 0.2.0-rc.2，**需重启生效**。
- **dsh-ops 0.3.0**：已装已生效（走 bundle 层，在 cordis.yml 非 cordis.patch.yml）。
- **KV Cache 视频**：DeepSeek 官方上下文硬盘缓存，命中 0.1 元/百万tokens vs 未命中 1 元（差 10 倍）；dsh-ops 用 Rust 重写底层工具 + 换掉 PowerShell 终端。
- **agent-teams**：已挂载（1796 行 insert）。用法：当前会话变队长，建队/拆任务/派活/收结果；成员只能是新建子 Agent，不能收编已有对话。
- **关键路径**：harness `%APPDATA%\dsh-desktop\harness\`；会话 `harness\sessions\`（解压用 profiles/web/node_modules/fzstd）；日志 `%APPDATA%\dsh-desktop\logs\harness.log`（UTC）；配置文件 `profiles\web\package.json` + `cordis.patch.yml`。
- **asar 包读取踩坑**：`@deepseek-ai/dsh` 等是 unpacked 或 offset 为 null；叶子节点结构是 `{size, offset}`（offset 为字符串）直接取；脚本被 stray 字符污染过多次 → 用 Write 工具写脚本文件，避免内联转义。

---

## 八、待办

1. **配置默认 Google CSE 引擎**（task #3，长期 pending）：schemastery 已是 3.18.4 ≥3.18.3 满足；字段 `googleApiKey`（env GOOGLE_CSE_API_KEY）、`googleSearchEngineId`（env GOOGLE_CSE_ID）；需用户 CSE ID + API key。
2. **重启 DSH 让 dsh-task-notify 生效**并测 Windows 通知。
3. **computer use**（待办，观望中）：路径 A PyAutoGUI+mss（推荐基底）/ B Playwright / C 完整框架（等模型商稳定）；待用户定方案和测试范围。
4. **verify compaction-image-offload**（task #7）：已确认内置不可手动挂，且内置下 AMD 仍 413，需换思路（压缩/换 provider）。
5. **nvidia 断连**：靠网络间歇恢复/切 Clash 节点；caw/mt 需充值。
6. **pnpm overrides**：防止 web-ui-all 依赖锁 0.3.6 把皮肤/UI 插件降级。
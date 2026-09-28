# dsh 0.1.7-rc.2 调研：破坏性变更、browser 不兼容证据、插件判定表

调研日期：2026-09-27/28
对象：`@deepseek-ai/dsh` 0.1.7-rc.2（隔离试装，见 handoffs/2026-09-27-dsh-017-trial.md）
状态：**调研中，未切主环境**

## 1. 破坏性变更要点

### 1.1 settings 迁移机制（新增）
0.1.7 引入 `importLegacyDocument()`，源码位于
`node_modules/@deepseek-ai/dsh-settings/lib/index.js:343-363`：

```js
async importLegacyDocument() {
  const profile = this.ownerContext.profileContext;
  const path = join(profile.home, "settings.yaml");   // profile.home，非 DSH_HOME 根
  if (!existsSync(path)) return;                       // 文件不存在 → 直接 return
  const imported = `${path}.imported`;
  await rename(path, imported);                        // 先改名，防重复导入
  const sections = parse(await readFile(imported, "utf8"));
  for (const [section, values] of Object.entries(sections ?? {})) {
    const ns = LEGACY_SECTION_ENTRIES[section] ?? section;
    try { await this.update(ns, values); }
    catch (error) { logger.warn("settings: section %s of %s was not imported into entry %s", ...); }
  }
  logger.info("settings: imported %s into profile %s", imported, profile.name);
}
```

关键点：
- 迁移读取路径是 **`profile.home/settings.yaml`**（= `DSH_HOME/profiles/<name>/settings.yaml`），
  **不是** `DSH_HOME/settings.yaml`。
- 迁移成功会打 `settings: imported ...` info；单段失败打 warn。**没有这些日志 = 迁移未执行**。
- 迁移产物的文件名后缀也是 `.imported`（先 rename 再 parse），因此磁盘上出现 `.imported`
  不能反推迁移跑过——需核对日志。

### 1.2 端口 / 网络
- 无直接破坏性变更；但 Windows 保留端口段会直接导致 `EACCES`（见第 3 节）。

## 2. dsh-browser 不兼容证据

**结论：0.1.7 下 dsh-browser 不可用，必须等官方修复。**

- 上游 issue/PR：**#102 / #103**（browser 桥相关的兼容性问题，PR#103 为修复）。
- 决策：**不手动打 PR#103**，等官方合并（理由见 tech-notes/decisions.md）。
- 影响面：browser profile（3081）继续跑在 0.1.5-rc.3，不随主环境升级。

## 3. 17 个插件在 0.1.7 下的判定表

| # | entry | 0.1.5 状态 | 0.1.7 判定 | 依据 |
|---|---|---|---|---|
| — | （待逐项实测填充） | — | — | 本轮未逐项实测 |

> ⚠️ **本节为占位**。本轮试装因 webserver 起不来（EACCES）+ 工具调用格式损坏而中止，
> **未完成 17 个 entry 的逐项判定**。判定表须在端口修复后重跑 017 环境补齐。
> 已知禁用集合仍以 tech-notes/known-facts.md 与 INDEX.md 的 17 entry 清单为准。

## 4. 本轮环境事实（供复现）

- 试装目录 `E:\work\dsh-run-017`（独立 node_modules），DSH_HOME `E:\work\dsh-home-017`。
- profile `diag-min`，端口 **3400**（原 3190 落 Windows 保留段 3135–3234 → `EACCES`）。
- 启动脚本 `E:\work\dsh-run-017\start-diag-min-017.cmd`。
- 迁移未执行 → 默认 provider 回落 `deepseek-official` → 请求 401（该 key 已失效，见 known-facts.md）。

## 5. 迁移机制复核与手工兜底（2026-09-28 会话）

### 5.1 七条复核结果（每条附命令与原始输出）

| # | 命令（cwd = `E:\work\dsh-run-017\node_modules\@deepseek-ai`） | 原始输出要点 | 标注 |
|---|---|---|---|
| 1 | `grep -n "resolveDshHome\|DSH_HOME" dsh-home-paths/lib/index.js` | `15:const DSH_HOME_ENV = "DSH_HOME";`、`73:function resolveDshHome(configured, env = process.env) {`、`74: const fromEnv = env[DSH_HOME_ENV];` | 【复核通过】 |
| 1b | `sed -n '73,77p' dsh-home-paths/lib/index.js` | `return resolve(expandHomePath(configured ?? (fromEnv !== void 0 && fromEnv.trim().length > 0 ? fromEnv : defaultDshHome())));` | 【复核通过】 |
| 2 | `grep -n "PROFILES_DIR\|resolveProfileDir" dsh-app-boot/lib/index.js` | `485:const PROFILES_DIR = "profiles";`、`524:function resolveProfileDir(name, home = resolveDshHome()) {`、`526: return join(home, PROFILES_DIR, name);` | 【复核通过】 |
| 3 | `grep -n "importLegacyDocument\|settings.yaml\|\.imported\|loader.await" dsh-settings/lib/index.js` | `339: ctx.root.loader.await().then(() => this.importLegacyDocument())`、`346:async importLegacyDocument() {`、`348: const path = join(profile.home, "settings.yaml");`、`350: const imported = ${path}.imported;` | 【复核通过】 |
| 4 | `grep -n "harness home\|settings.yaml" dsh-settings/README.md` | `35:` 原文 "a `settings.yaml` left in the **harness home** by earlier releases is imported once" | 【复核通过】（与代码矛盾） |
| 5 | `grep -rn "profileContext" dsh/lib/` | `profile-boot-BZ2ZjNWi.js:259`、`:271`、`:273:hostCtx.provide("profileContext", profileContext);` | 【复核通过】 |
| 6 | `grep -n -B2 "profileContext" dsh-base/cordis.patch.yml` | 4 处 disabled：`20-22 plugin-manager`、`28-30 hmr`、`97-99 config-editor`、`101-103 settings`；另 `113-115` 一处非 disabled | 【复核通过】 |
| 7 | `--dump-config \| grep -n -B2 "profileContext"` | （空） | **【已撤回：执行异常】** |

### 5.2 第 7 条的撤回与对照（重要）

**原说法**：第 7 条 `--dump-config | grep -n -B2 "profileContext"` 输出为空，据此曾推断 dump 组合树中无 profileContext 字样。

**复核结果**：同一份 dump 落盘后重查，`profileContext` 实际出现 **7 次**。第 7 条的空输出是管道调用返回的**假空结果**，不是事实。

**对照证据（dump 落盘重查）**：

| # | 命令 | 原始输出 |
|---|---|---|
| 补1 | `node .../dsh/lib/bin.js --profile diag-min --dump-config > /e/work/dump-017.txt; echo "exit=$?"` | `exit=0` |
| 补2 | `wc -l /e/work/dump-017.txt` | `1252 /e/work/dump-017.txt` |
| 补3 | `grep -n -A4 "id: settings$" /e/work/dump-017.txt` | `61:- id: settings` / `62-  name: '@deepseek-ai/dsh-settings'` / `63-  disabled: !!js '!ctx.get(''profileContext'')'` |
| 补4 | `grep -c "disabled" /e/work/dump-017.txt` | `26` |
| 补5 | `grep -c "profileContext" /e/work/dump-017.txt` | `7` |

**教训（已写入执行规范）**：结论与已有证据冲突时，先把输出落盘成文件，再在文件上重查，不直接采信管道输出。

### 5.3 dump 中 profileContext 的 7 处分布与"4 处"更正

`grep -n -B2 "profileContext" /e/work/dump-017.txt` 原始输出显示 7 处：

| dump 行 | entry id | 表达式 |
|---|---|---|
| 5-7 | `plugin-manager` | `disabled: !!js '!ctx.get(''profileContext'')'` |
| 10-12 | `hmr` | 同上 |
| 58-60 | `config-editor` | 同上 |
| 61-63 | `settings` | 同上 |
| 68-70 | （`desktopPlatform` config） | `ctx.get('profileContext')?.name === 'desktop'`（非 disabled） |
| 524-526 | `ui-sidebar-browser` | `disabled: !!js ctx.get('profileContext')?.name !== 'desktop'`（另一表达式） |
| 1244-1246 | `tool-plugin-manager` | `disabled: !!js '!ctx.get(''profileContext'')'` |

**同表达式 `disabled: !!js '!ctx.get(''profileContext'')'` 共 5 处**：

- 【复核通过】**顶层 4 处**：`plugin-manager`(5-7)、`hmr`(10-12)、`config-editor`(58-60)、`settings`(61-63)
- 【部分更正】**另有嵌套子 entry `tool-plugin-manager`**（dump 1244-1246，源文件以 `disabled: true` 硬编码，经 patch 组合后展开为同一表达式），此前漏报 → **合计 5 处**
- 【复核通过】dump 第 61-63 行 `settings` + disabled 表达式，与 Step 4 说法一致

### 5.4 根因表述（严格按证据分级）

- **迁移未执行【实测】**：`settings.yaml` 未被改名（无 `.imported`）、`--dump-config` 中 `agent-default-model` 为默认值 `deepseek-official`/`deepseek-flash`、rpc-test 返回 401（key `****a8a1`，已失效）。三项独立证据一致。
- **settings 等 entry 在 dump 中挂着 `!ctx.get('profileContext')` 禁用条件【复核通过】**（见 5.3）。
- **运行时该条件是否为真、settings 是否确实未激活——【未证实】**。
  - 下个会话验证方法：通过设置接口是否存在来判断（若设置页/插件管理器 API 不可访问，则确证被禁用）。
- **"profileContext 时序导致 entry 禁用"目前是【假设】**，依据只有源文件里的表达式（第 6 条）与 dump 分布（5.3），**无运行时证据**。

### 5.5 【假设】desktop profile 可能绕开该问题

dump 第 68-70 行（`desktopPlatform`）与第 524-526 行（`ui-sidebar-browser`）出现 `profileContext?.name === 'desktop'` 判断，**推测 0.1.7 预设了名为 `desktop` 的 profile，桌面端宿主可能会提供 profileContext，从而绕开 settings 被禁用的问题。**

- 状态：**【假设】**，待装桌面端时验证。
- 验证方法：看设置页和插件管理器能不能用。

### 5.6 手工兜底（部分生效，链路未跑通）

**操作**：在 `profiles\diag-min\cordis.patch.yml` 只写 `agent-default-model` 一段（provider=nvidia, model=deepseek-ai/deepseek-v4.1-flash），**不写 `llm-pi-ai`**。

patch 全文（177 B / 6 行，grep 校验无 key 特征，计数 0）：

```yaml
- insert:
    - id: agent-default-model
      name: '@deepseek-ai/dsh-agent-default-model'
      config:
        provider: nvidia
        model: deepseek-ai/deepseek-v4.1-flash
```

**结果【实测】**：报错从 `AUTH 401 (****a8a1)` 变为

```
TURN/END: {"kind":"error","error":{"message":"no adapter registered for provider \"nvidia\"","code":"NO_ADAPTER"}}
RESULT: ERROR
--- 退出码: 1 ---
```

**结论**：patch 生效（provider 已切到 nvidia），但 **0.1.7 需要在 `llm-pi-ai` 里声明 provider 才会注册 adapter**；只补 `agent-default-model` 不足以跑通。

### 5.7 settings.yaml 结构与安全规则

`profiles\diag-min\settings.yaml`（3765 B，复制自主环境）顶层段（`grep -nE "^[A-Za-z0-9_-]+:"`）：

```
1:ui-onboarding:   3:llm-pi-ai:   72:agent-default-model:   75:pet:
81:skin-background:  86:skin-wallpaper:  93:task-board:  95:desktop-launcher:
97:dsh-better-sidebar:  100:ui-theme:  102:web-search-deepseek:  104:agent-loop:
106:skin-custom-theme:  108:approve-for-me:  135:mnemon:
```

`llm-pi-ai.providers` 下的四个 provider 段（只记结构，不记值）：

| provider | 字段 |
|---|---|
| `amd` | `apiKeyEnv`, `api`, `baseURL`, `models`(4) |
| `huggingface` | `apiKeyEnv`, `models`(2) |
| `mt` | `displayName`, `apiKeyEnv`, `api`, `models`(...) |
| `nvidia` | `apiKeyEnv`, `models`(6) |

**安全规则【实测】**：

> **复制到 `profiles\diag-min\settings.yaml` 的整份文件禁止进入仓库**（含 1 处疑似 key）。不跟行号挂钩。

补充事实：疑似 key 的 grep 计数为 **1**，位于第 **93** 行，顶层段名为 **`task-board:`**（已由 `grep -nE "^[A-Za-z0-9_-]+:"` 坐实，93 行即该段起始行），**在 `llm-pi-ai` 段（3-71 行）之外**。CRLF 计数为 0（纯 LF）。

### 5.8 待办

1. 确认 0.1.7 的 `llm-pi-ai` schema 是否支持 `apiKeyEnv` 引用（读 `dsh-llm-pi-ai` 源码里注册 adapter 的地方，确认最小字段集）。
2. 若支持，则追加只含 `nvidia` 一个 provider 的最小 `llm-pi-ai` 段，key 用 `apiKeyEnv: NVIDIA_API_KEY` 引用，不写明文；然后跑 rpc-test。
3. 通过设置接口是否存在，验证 settings entry 运行时是否真被禁用。
4. 补齐 0.1.7 下 17 个插件的逐项判定表。

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

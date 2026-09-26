# Runbook：记忆仓库维护（本仓库）

## 结构与用途
- `INDEX.md`：入口——当前目标、进度、文件索引、已证伪清单。
- `tech-notes/`：技术结论与原因（升级、插件兼容、profile、bridge 安全、genspark、硬规则）。
- `runbooks/`：可执行步骤（启动 profile、升级 dsh、记忆仓库维护、bridge 补丁）。
- `handoffs/`：每次对话结束时的交接摘要，`YYYY-MM-DD-*.md`。

## 工具
- gitleaks 8.30.1：`.tools/gitleaks/gitleaks.exe`（二进制不入库，见 .gitignore 的 `.tools/`）。
- pre-commit hook：`.githooks/pre-commit`，已 `git config core.hooksPath .githooks` 全局指向本仓库。
  - gitleaks `protect --staged` 检测。
  - 自定义扫描：拒绝暂存区文本中的凭据样式——会话 ID 查询参数、URL 令牌参数、NVIDIA 密钥前缀、长 `Bearer` 授权头（规则字面量见 `.githooks/pre-commit`，该文件自身被扫描豁免）。

## 提交流程
1. 本地写完 → 自查无 key/token/日志原文。
2. `git add` → 提交（hook 自动拦截）。
3. 推送前人工复查 diff；不推任何含敏感字的文件。

## 写记忆准则
- 每条结论必须带"原因"或"证据出处"。
- 不写 key、token、cookie、日志原文（可写"长度/格式/存在性"级信息）。
- 已证伪结论进 INDEX.md 的"已证伪"清单，不反复复勘。
- 用户口述但与本地证据不符/缺失时，标注"来源=用户口述，本地无独立证据"。
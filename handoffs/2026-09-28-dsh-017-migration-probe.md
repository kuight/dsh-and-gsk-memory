# Handoff：2026-09-28 dsh 0.1.7-rc.2 隔离试装（017）迁移机制复核

## 环境
- 试装运行目录：`E:\work\dsh-run-017`（独立 node_modules）
- 独立 DSH_HOME：`E:\work\dsh-home-017`
- profile：diag-min，端口 **3400**（3190 落 Windows 保留段 3135-3234，`EACCES`）
- 启动脚本：`E:\work\dsh-run-017\start-diag-min-017.cmd`（注释已从 3190 改为 3400）
- **017 实际启动方式：直接 exec**（export DSH_HOME + PATH + NO_PROXY，再跑 node bin.js），不用 .cmd
- 3400 服务当前状态：**PID 31984，保留未停**

## 本会话结论

### 1. 迁移未执行【实测】
三项独立证据一致：
- `profiles\diag-min\settings.yaml`（3765 B）**未被改名**成 `.imported`，目录内无 `.imported`
- `--dump-config` 中 `agent-default-model` 为**默认值** `deepseek-official`/`deepseek-flash`；`llm-pi-ai` entry 存在但**无 config 段**
- rpc-test（3400）返回 `AUTH 401`，key `****a8a1`（已失效的 deepseek-official key）

### 2. 根因：未证实
- 【复核通过】settings 等 entry 在 dump 中挂着 disabled 条件（`!ctx.get('profileContext')`），顶层 4 处 + 嵌套 `tool-plugin-manager` 1 处 = **5 处**（dump 第 5-7、10-12、58-60、61-63、1244-1246 行）
- 【假设】"profileContext 时序导致 entry 禁用"——只有源文件与 dump 依据，**无运行时证据**
- 下会话验证：通过设置接口是否存在判断（设置页/插件管理器 API 不可访问 = 确证被禁用）

### 3. 手工兜底（部分生效，链路未跑通）
只写 `agent-default-model`（provider=nvidia）→ 报错从 `401` 变为：
`NO_ADAPTER: no adapter registered for provider "nvidia"`
→ patch 生效，但 **0.1.7 需在 `llm-pi-ai` 里声明 provider 才会注册 adapter**

### 4. 关键更正
- 【部分更正】顶层 4 处【复核通过】（plugin-manager、hmr、config-editor、settings），漏报嵌套的 `tool-plugin-manager`，**合计 5 处**
- 【已撤回：执行异常】dump 管道 grep 返回假空输出 → 落盘后重查为 **7 处**

## 关键事实（详见 tech-notes）
- `profile.home` = `DSH_HOME/profiles/<name>`（三处行号：dsh-home-paths:73-76、dsh-app-boot:524-527、dsh-app-boot:485）
- README:35（harness home）与 lib/index.js:348（profile.home）矛盾，**以代码为准**
- 日志只在 StartupError 时落盘；正常启动看 stderr + cordis.yml 是否被重写
- Git Bash 下 .cmd 会被拆坏；3190 端口坑；`~/.dsh` 无 logs 目录可作隔离证据

## 安全
- **`profiles\diag-min\settings.yaml` 整份文件禁止进仓库**（含 1 处疑似 key，第 93 行，段名 `task-board:`）
- `E:\work\dump-017.txt` 留在 E:\work，不进仓库

## 待办（下会话）
1. 确认 0.1.7 的 `llm-pi-ai` schema 是否支持 `apiKeyEnv`（读 dsh-llm-pi-ai 源码注册 adapter 处）
2. 若支持，追加只含 `nvidia` 的最小 `llm-pi-ai` 段（key 用 `apiKeyEnv: NVIDIA_API_KEY`，不写明文），跑 rpc-test
3. 通过设置接口验证 settings entry 运行时是否真被禁用
4. 补齐 17 个插件判定表
5. 【假设待验】desktop profile 可能绕开 profileContext 问题（dump 68-70、524-526 行有 `name === 'desktop'` 判断）

## 硬约束（沿用）
- 不读取 `.credentials.yaml` 内容；不动 3080/3081/3180；不改 `~/.dsh` 和 `E:\work\dsh-run`
- 汇报不超 40 行；根目录与 `~/.dsh` 下禁止递归搜索

## 执行者须知（2026-09-28 本会话教训）

- 本会话多次出现 **"写入未生效却回显成功"**：命令输出看似成功（甚至回显了行数），但复核发现文件根本没变。
- **每次写入后必须单独复核**：用 `wc -l` 或 `grep -c <新内容标记>` 确认改动真的落盘，不能只信写入命令的回显。
- **结论与已有证据冲突时**，先把输出落盘成文件，再在文件上重查，不直接采信管道输出（反例：dump 管道 grep 曾返回假空结果，落盘后重查为 7 处）。
- 含反引号的内容不要直接拼进 bash 双引号命令（shell 会执行反引号内命令，导致内容被吃掉）；用 sed 按行号替换或文件写入方式。

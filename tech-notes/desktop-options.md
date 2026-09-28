# DSH 桌面端选项调研（2026-09-28）

## 1. 官方版

- 下载：`download.deepseek.com`
- 内置版本：**0.1.7-rc.1**，可自更新到 **rc.2**
- 前置要求：充值实名 或 DeepSeek key
- 来源：36氪 / AI前线报道

## 2. 社区版 DSH Desktop

- 仓库：`github.com/dataelement/dsh-desktop`
- 基于 **0.1.7-rc.1**
- 特性：Safe Mode、插件恢复界面
- 数据位置：Electron 用户目录
- ⚠️ **手机配对会走公网隧道** —— 不用就别开

## 3. anywhere-labs 版

- 号称与命令行版**配置互通**
- ⚠️ 风险：会读写 `~/.dsh`

## 4. 装之前的保护步骤

1. 停 3400
2. 备份 `~/.dsh`（**不含** `.credentials.yaml` 和 `node_modules`）
3. 生成 `snapshot-pre-desktop.txt`
4. 首次启动后**先不配模型**，只查：
   - 数据目录
   - 端口
   - `~/.dsh` 的变化
5. **有变化就停**

## 5. 待验证（两项）

1. **desktop profile 能不能绕开 profileContext 问题** —— 看设置页和插件管理器能否使用。
2. **从图标启动时会不会继承 HTTPS_PROXY**。

# Runbook：启动各 dsh profile

所有脚本统一模式：PATH 前置 `F:\Apply\node24`（满足 engines）+ 设置 `NO_PROXY=...,integrate.api.nvidia.com`（绕过 7897 代理卡 NVIDIA 的坑）+ `--no-open`。
原因：两个环境变量缺一不可——Node 版本不符会启动失败；NO_PROXY 缺失会让模型请求走代理慢/挂（实证：349s vs 11s）。

## web（主 profile，端口 3080）

```
E:\work\dsh-run\start-dsh.cmd
```

- 用途：日常主工作区。
- 当前注意：28 个 entry 被临时禁用（诊断期），turn/end 报错支线未查完，见 tech-notes/plugin-compat-issues.md。

## diag-min（最小诊断 profile，端口 3180）

```
E:\work\dsh-run\start-diag-min.cmd
```

- 用途：**模型链路诊断基准**（只含 base + web-app）。
- 方法论：升级/排查模型问题先在此 profile 验证"模型能回复"，通过后再谈第三方插件。

## browser（dsh-browser 桥，端口 3081）

```
E:\work\dsh-run\start-browser.cmd
```

- 用途：跑 `@yuxianglin/dsh-bridge-browser`，供 Chrome 扩展连接。
- 端口 3081 在扩展自动发现列表内 → 扩展零配置连接，无需手动填地址/token。
- dsh 重启后扩展会重连；token 为回环免 token（chrome-extension:// Origin），无需更新扩展设置。

## 验证命令

```
netstat -ano | findstr ":3081"          # 确认端口监听
curl http://127.0.0.1:3081/ext/bridge-config   # 桥是否就绪（返回 wsUrl）
```

## 启动后自查清单

1. 端口已监听、日志有 `dsh web: http://...` 行
2. `curl /ext/bridge-config` 返回 wsUrl（browser profile）
3. 发一句话验证模型回复时长（应 ~11s 级，若 ~350s 查 NO_PROXY）
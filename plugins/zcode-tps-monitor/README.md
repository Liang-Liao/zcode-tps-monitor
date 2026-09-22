# zcode-tps-monitor

项目介绍、安装与使用说明见[仓库首页 README](../../README.md)。本文件面向插件内部结构与开发测试。

## 能力一览

| 形态 | 入口 | 说明 |
|---|---|---|
| 本轮即时速率 | `hooks/stop.mjs` | 回复刚结束(Stop 钩子)时按最新 `turn_id` 圈定本轮全部请求,经 `systemMessage` 直接显示本轮即时速率行;可经 `~/.zcode/tps-monitor.config.json`(`{"stopHookLine": false}`)停用并回到模型转发旧行为 |
| 上下文注入 | `hooks/prompt-submit.mjs` | 每轮对话读取 ZCode usage 数据库,注入上一轮速率作为模型上下文;`{"tokenRateLine": false}` 可关闭 |
| 会话提示 | `hooks/session-start.mjs` | 会话启动时记录会话 ID,并注入一行使用提示 |
| 自检 | `/tps-doctor`(`scripts/doctor.mjs`) | 检查 Node 版本、数据库与表结构、状态/配置文件;`--json` 可编程消费 |

## 数据源

### Token 速率(真实,默认开启)

由钩子读取 ZCode usage 数据库(`model_usage` 表)计算,可手动验证:

```bash
node scripts/token-rate.mjs            # 人类可读
node scripts/token-rate.mjs --turn     # 本轮(刚结束轮次)即时速率
node scripts/token-rate.mjs --json     # JSON
```

可设置 `ZCODE_SESSION_ID` 环境变量只统计当前会话(钩子已自动设置)。

数据库路径默认按用户主目录解析(`~/.zcode/cli/db/db.sqlite`,Windows 同理),可用 `ZCODE_USAGE_DB` 环境变量覆盖;以只读方式打开 WAL 库,不影响运行中的客户端。

## 开发与测试

```bash
# Token 速率 CLI(手动验证钩子同源数据)
node scripts/token-rate.mjs            # 人类可读
node scripts/token-rate.mjs --turn     # 本轮(刚结束轮次)即时速率
node scripts/token-rate.mjs --json     # JSON

# 自检
node scripts/doctor.mjs             # 人类可读(❌ 项给出修复建议)
node scripts/doctor.mjs --json      # JSON,失败时退出码 1

# 单元测试(仓库根目录;临时库夹具,不读真实数据)
node --test
```
## 目录结构

```
zcode-tps-monitor/
├── .zcode-plugin/plugin.json   # 插件清单(name / 描述 / 命令注册)
├── .claude-plugin/plugin.json  # 兼容清单
├── commands/tps-doctor.md      # /zcode-tps-monitor:tps-doctor
├── hooks/hooks.json            # 钩子注册(SessionStart + UserPromptSubmit + Stop)
├── hooks/session-start.mjs     # 会话启动:记录会话 ID + 使用提示
├── hooks/prompt-submit.mjs     # 每轮:注入上一轮速率作模型上下文
├── hooks/stop.mjs              # 回复结束:按 turn_id 圈定本轮,systemMessage 显示即时速率
├── scripts/
│   ├── token-rate.mjs          # token 速率 CLI(人类可读 / --json)
│   └── doctor.mjs              # 自检(人类可读 / --json)
└── docs/
    └── effect-token-rate.png   # 效果截图
```

## 修改后生效

在 **设置 → 插件管理** 中重新安装/刷新插件,并重开会话使钩子重新注册。要求 Node ≥ 22.5(需内置 `node:sqlite`)。

## License

[MIT](../../LICENSE)

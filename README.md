<img src="assets/icon.png" align="right" width="96" alt="zcode-tps-monitor 图标">

# zcode-tps-monitor

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Node](https://img.shields.io/badge/node-%E2%89%A5%2022.5-brightgreen)
![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20macOS%20%7C%20Linux-blue)

**ZCode 会话级 Token 速率监控插件。** 每轮回复结束时自动显示**本轮即时** tok/s —— 数据直接读取 ZCode usage 数据库,非模型自述、非估算。

> 本仓库同时是一个 ZCode 本地插件市场(marketplace 名称:`tps-local-marketplace`),插件本体位于 [`plugins/zcode-tps-monitor/`](plugins/zcode-tps-monitor/README.md)。

## 效果预览

每轮回复结束时自动显示一行速率指标,无需任何手动操作。行在回复刚结束的瞬间采样(Stop 钩子),头条就是**本轮的即时速率**——多段工具调用的长轮次按"总产出 / 总生成时长"加权:

![token 速率行效果](plugins/zcode-tps-monitor/docs/effect-token-rate.png)

| 字段 | 含义 |
|---|---|
| `537.3 tok/s` | 本轮即时输出速率(含思考 token;多段轮次为加权速率) |
| `首字 3.0s` | 首 token 延迟(TTFT,本轮第一段) |
| `输出 223 tok / 生成 0.4s` | 本轮输出 token 数与纯生成耗时(不含段间工具等待) |
| `2 段 / 峰 537.3` | 本轮的请求段数与单段峰值速率(多段轮次才显示) |
| `近3次均 494.9` | 最近数轮滑动平均 |
| `累计 51.3k tok` | 当前会话累计输出(独立统计,不受窗口限制) |
| `⏱ 10:23:04` | 采样时刻(回复结束时间) |

数字显示规则:每轮「输出」用千分位精确数字(如 `2,762 tok`);「累计」用紧凑单位——千以下原始、1k~1万一位小数(`9.8k`)、1万~100万取整(`51k`)、百万以上一位小数 M(`73.8M`)。

## 功能特性

- **真实 Token 速率注入(默认开启)** —— 每轮回复结束时自动显示本轮即时 tok/s(含思考 token)、首字延迟、输出 token 数、生成耗时、段数/峰值与会话累计;下一轮提问时,上一轮速率作为上下文注入
- **环境自检** —— `/tps-doctor` 逐项排查 Node 版本、usage 数据库、状态文件与配置,速率行不见了?一条命令定位
- **纯本地运行** —— 只读 ZCode usage 数据库,无任何对外网络访问,无遥测、无外部依赖

## 安装

### 方式一:从 GitHub 添加(推荐)

在 ZCode 中执行:

```text
/plugin marketplace add shy3130/zcode-tps-monitor
/plugin install zcode-tps-monitor@tps-local-marketplace
```

### 方式二:本地目录

克隆本仓库后,在 ZCode 中打开 **设置 → 插件管理 → 发现 → +**,来源选择"本地目录",指向仓库根目录即可。

### 更新

```text
/plugin marketplace update tps-local-marketplace
```

更新后重装/升级插件,并重开会话使钩子重新注册。

## 使用

| 场景 | 操作 |
|---|---|
| 查看每轮速率 | 无需操作,每轮回复结束时自动显示本轮速率 |
| 关闭本轮即时行 | `~/.zcode/tps-monitor.config.json` 写入 `{"stopHookLine": false}`,重开会话生效 |
| 关闭全部速率注入 | 同文件写入 `{"tokenRateLine": false}`,重开会话生效 |
| 环境自检 | 输入 `/tps-doctor` 逐项排查 |

要求 Node ≥ 22.5(需内置 `node:sqlite`,Windows / macOS / Linux 相同)。

## 工作原理

```
用户发送消息
   │
   ▼
UserPromptSubmit 钩子
   │  读取 ZCode usage 数据库,注入上一轮速率作模型上下文
   ▼
模型回复(工具调用 × N 段)
   │
   ▼
Stop 钩子(回复刚结束,本轮已全部入库)
   │  按最新 turn_id 圈定本轮全部请求,
   │  计算即时速率(总产出 / 总纯生成时长)
   ▼
systemMessage 直接显示本轮速率行
```

- **SessionStart 钩子**:会话启动时记录当前会话 ID 并注入使用提示
- **UserPromptSubmit 钩子**:每轮触发一次,单次为毫秒级数据库读取,开销可忽略;此刻本轮尚未发生,因此只注入上一轮数据作上下文
- **Stop 钩子**:回复刚结束、本轮数据已完整入库的瞬间触发,按 `turn_id` 精确圈定本轮(一次用户消息触发的全部请求,含多段工具调用),经 `systemMessage` 由客户端直接显示——无需模型转发,天然零滞后

## 常见问题

**Q:显示的速率准确吗?**

A:速率由 ZCode usage 数据库中的真实 token 累计值计算得出,口径为模型输出侧 token。注意:行在发送消息瞬间采样,显示的是上一条已完成回复的速率;当前回复的速率会在下一轮显示。与其他工具显示的统计数字可能因统计窗口不同而略有差异。

**Q:速率行突然不见了?**

A:运行 `/tps-doctor` 自检。常见原因:Node 版本低于 22.5(需内置 `node:sqlite`)、ZCode 更新后表结构变化、升级插件后未重开会话(钩子需新会话注册)、或配置文件里关闭了注入。

**Q:macOS / Linux 支持吗?**

A:支持。钩子、命令均为跨平台 Node 实现;usage 数据库路径按用户主目录自动解析(`~/.zcode/cli/db/db.sqlite`),特殊安装位置可用 `ZCODE_USAGE_DB` 环境变量覆盖。

**Q:插件会联网吗?**

A:不会。插件只读取本机 usage 数据库(只读)并在本地生成速率行,没有任何对外网络请求。

## License

[MIT](LICENSE) © 2026 shy3130

# Agent Logging Standard — Agent Reference

⚠️ AI AGENTS: Before writing any logging code, read AGENT_LOGGING_STANDARD_zh.md in full. This README is only a summary.

## 这是什么？（面向 AI Agent，非面向人类用户）

本项目是专供 **AI 编码 Agent**（Claude Code、GPT 等）内化使用的日志编程规范参考包，目标是让 Agent 产出的日志代码开箱即用、出问题能快速定位。规范萃取自 [OptoSync](https://github.com/YoukoSaint/OptoSync) 项目的生产实践。

## 使用方式

### 方式一：Claude Code Skill（首选 Preferred）

```bash
cp SKILL.md ~/.claude/skills/agent-logging-standard/SKILL.md
```

安装后在 Claude Code 中输入 `/agent-logging-standard` 即可激活。**这是 Claude Code 用户的推荐方式**——Skill 会自动将完整规范注入 Agent 上下文。

### 方式二：直接注入（通用方法）

**将 AGENT_LOGGING_STANDARD_zh.md 全文作为 system prompt 喂给 Agent。不要只给摘要。**

全量注入才能覆盖级别决策速查表、格式字段说明和仪器传输层日志章节。仅给 README 摘要会导致 Agent 遗漏关键实现细节。

## 文件说明

| 文件 | 类型 | 语言 | Required reading? | 说明 |
|------|------|------|-------------------|------|
| `SKILL.md` | Claude Code Skill | EN | Reference | 可安装到 `~/.claude/skills/`，Agent 激活后自动加载完整规范 |
| `AGENT_LOGGING_STANDARD_zh.md` | 完整规范 | 中文 | **YES** | 完整规范：级别定义、结构化模式、运维模式、性能、错误处理、安全合规、反模式、仪器传输层日志、决策速查表 |
| `AGENT_LOGGING_STANDARD.md` | 完整规范 | EN | Reference | 同上英文版（不含速查表和传输层章节） |

## 规范概要

```
DEBUG   → 开发诊断（协议细节、变量值、函数进出、传输层原始字节）
INFO    → 运行事件（启动/停止、配置变更、操作完成，必须带指标）
WARNING → 可恢复异常（自动降级、重试、资源趋近上限）
ERROR   → 操作失败（请求失败、数据损坏、外部服务不通，含堆栈）
CRITICAL → 系统崩溃（OOM、磁盘满、安全入侵，必须含恢复建议）
```

### 核心设计理念

- **双 Handler** — 文件（DEBUG + 行号）用于深度排查，控制台（INFO 简洁）用于实时观察
- **时间戳文件** — 每次运行生成 `AppName_YYYYMMDD_HHMMSS.log`，互不污染
- **热路径克制** — >100 Hz 的循环内不打日志，高频事件用取模采样
- **安全红线** — 密码、令牌、PII 绝不写入日志
- **传输层标记** — USB/TCP/Serial/GPIB 通信日志带 `[transport=xxx][addr=xxx]` 标签
- **发现日志** — 仪器自动发现阶段每条传输通道的扫描结果（含负结果）
- **连接生命周期** — 连接/断开/重连事件全链路记录，含传输层指标

## 13 项 Agent 检查清单

> 详细实现参见 AGENT_LOGGING_STANDARD_zh.md 各章节。

### 基础日志配置（§3 结构化日志模式）

- [ ] `setup_logging()` 在入口点调用，使用时间戳文件
- [ ] 文件 Handler = DEBUG，控制台 Handler = INFO
- [ ] 每个模块 `logging.getLogger(__name__)`

### 日志内容与格式（§4 运维日志模式, §6 错误处理与日志, §8 指标与可观测性）

- [ ] 长运行操作记录 START/STOP + 指标
- [ ] except 块中使用 `logger.exception()`
- [ ] 量化日志带单位（Hz、MB、ms、%）

### 性能与安全（§5 性能友好的日志, §9 安全与合规）

- [ ] 不在 >100 Hz 循环内打日志
- [ ] 使用 `%s` 格式化而非 f-string
- [ ] 绝不记录密码/令牌/PII

### 运维保障（§4.2 健康监控）

- [ ] 守护进程添加健康监控

### 仪器自动化补充（§14 仪器自动化传输层日志）

- [ ] 仪器日志标注传输层来源 `[transport=xxx][addr=xxx]`
- [ ] 独立子 Logger 分离 SCPI 协议日志与传输层日志
- [ ] 发现阶段记录每条传输通道的扫描结果

## 参考来源

- [OptoSync](https://github.com/YoukoSaint/OptoSync) — 多设备同步采集系统
- 核心参考文件：`core/logging_config.py`、`core/sync_controller.py`、`core/health_monitor.py`、`drivers/keithley_driver.py`

## 许可

MIT

# Agent Logging Standard

> 面向 AI 编码 Agent 的生产级日志编程规范，萃取自真实硬件采集项目的最佳实践。

## 项目简介

本项目提供一套专门为 **AI 编码 Agent**（Claude、GPT 等）编写的日志编程规范，目标是让 Agent 产出的代码具备 **生产运维可读性** —— 开箱即用、出了问题能快速定位。

规范基于 [OptoSync](https://github.com/YoukoSaint/OptoSync) 项目的日志实践提炼而成，结合运维思维做举一反三。

## 快速安装

### 方式一：Claude Code Skill（推荐）

```bash
# 安装为全局 skill（所有项目可用）
cp SKILL.md ~/.claude/skills/agent-logging-standard/SKILL.md
```

安装后在 Claude Code 中输入 `/agent-logging-standard` 即可调用。

### 方式二：直接引用

```bash
# 喂给 Agent 作为知识库
请遵循 AGENT_LOGGING_STANDARD_zh.md 中的日志规范，为以下模块编写代码……
```

## 文件说明

| 文件 | 类型 | 语言 | 说明 |
|------|------|------|------|
| `SKILL.md` | Claude Code Skill | EN | 可直接安装到 `~/.claude/skills/`，Agent 激活后自动执行 |
| `AGENT_LOGGING_STANDARD.md` | 参考文档 | EN | 完整规范：级别定义、结构化模式、运维模式、错误处理、安全合规、反模式 |
| `AGENT_LOGGING_STANDARD_zh.md` | 参考文档 | 中文 | 同上，追加级别决策速查表和日志格式字段说明 |

## 规范概要

```
DEBUG   → 开发诊断（协议细节、变量值、函数进出）
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

### 10 项 Agent 检查清单

- ✅ `setup_logging()` 在入口点调用，使用时间戳文件
- ✅ 文件 Handler = DEBUG，控制台 Handler = INFO
- ✅ 每个模块 `logging.getLogger(__name__)`
- ✅ 长运行操作记录 START/STOP + 指标
- ✅ except 块中使用 `logger.exception()`
- ✅ 量化日志带单位（Hz、MB、ms、%）
- ✅ 不在 >100 Hz 循环内打日志
- ✅ 使用 `%s` 格式化而非 f-string
- ✅ 绝不记录密码/令牌/PII
- ✅ 守护进程添加健康监控

## 参考来源

- [OptoSync](https://github.com/YoukoSaint/OptoSync) — 多设备同步采集系统（Keithley 2461 + Ideaoptics 光谱仪）
- 核心参考文件：`core/logging_config.py`、`core/sync_controller.py`、`core/health_monitor.py`、`drivers/keithley_driver.py`

## 许可

MIT

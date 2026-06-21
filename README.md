# Agent Logging Standard — Agent Reference

⚠️ AI AGENTS: Before writing any logging code, read AGENT_LOGGING_STANDARD.md and AGENT_LOGGING_STANDARD_zh.md in full. This README is only a summary.

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
- **维度化标签** — 每一行日志都按维度打标签便于分桶：
  - `USER_ACTION |` 用户交互（§15）
  - `HEALTH |` 系统资源 / 健康指标（§16）
  - `[transport=xxx][addr=xxx]` 物理 / 仪器接口（§14/§17）
  - `FILE_IO |` 文件与存储 I/O（§18）
- **传输层标记** — USB/TCP/Serial/GPIB 通信日志带 `[transport=xxx][addr=xxx]` 标签
- **发现日志** — 仪器自动发现阶段每条传输通道的扫描结果（含负结果）
- **连接生命周期** — 连接/断开/重连事件全链路记录，含传输层指标

## 25 项 Agent 检查清单

> 详细实现参见 AGENT_LOGGING_STANDARD_zh.md 各章节。

### 基础日志配置（§3 结构化日志模式）

- [ ] `setup_logging()` 在入口点调用，使用时间戳文件
- [ ] 文件 Handler = DEBUG，控制台 Handler = INFO
- [ ] 每个模块 `logging.getLogger(__name__)`
- [ ] 使用统一前缀便于 grep（`HEALTH |`、`USER_ACTION |`、`FILE_IO |`、`[transport=xxx][addr=xxx]`）

### 日志内容与格式（§4 / §6 / §8）

- [ ] 长运行操作记录 START/STOP + 指标
- [ ] except 块中使用 `logger.exception()`
- [ ] 量化日志带单位（Hz、MB、ms、%、°C、W）

### 性能与安全（§5 / §9）

- [ ] 不在 >100 Hz 循环内打日志
- [ ] 使用 `%s` 格式化而非 f-string
- [ ] 绝不记录密码/令牌/PII

### 用户交互（§15）

- [ ] 用户触发的所有事件含 `user_id`（不可逆 ID）
- [ ] 登录/登出/认证失败在 `audit.user` 子 logger 独立记录
- [ ] 配置变更记录 `old=` → `new=`（敏感字段 `[REDACTED]`）
- [ ] 权限拒绝事件含 `user_id` + `required_role` + `resource`

### 系统资源与周期性指标（§16）

- [ ] 守护进程添加健康监控
- [ ] HEALTH 行必含 `uptime / rss / cpu / thr / q` 五项
- [ ] 涉及 GPU 时记录 `gpu=idx:util|mem|temp|pwr`
- [ ] 高并发服务 HEALTH 含 `fds` + `net:est|tw|cw`
- [ ] 写盘型守护进程 HEALTH 含 `disk:mount=pct%`
- [ ] 阈值告警按 §16.4 表触发（WARNING/ERROR/CRITICAL）

### 物理接口（§14 / §17）

- [ ] 仪器日志标注传输层来源 `[transport=xxx][addr=xxx]`
- [ ] 独立子 Logger 分离 SCPI 协议日志与传输层日志
- [ ] 发现阶段记录每条传输通道的扫描结果
- [ ] 嵌入式/工业/视觉接口（§17）使用对应 transport 标签
- [ ] 高频物理接口（>100 Hz）用子 Logger + 周期汇总，禁止逐条

### 文件 I/O（§18）

- [ ] 文件 I/O 日志以 `FILE_IO | op=<op> kind=<kind> path=<path>` 开头
- [ ] 业务数据写入走 `tmp + os.replace` 原子写
- [ ] `fsync` 失败升 CRITICAL
- [ ] 写循环用 `rows % BATCH == 0` 取模聚合
- [ ] 配置加载失败 = CRITICAL

## 参考来源

- [OptoSync](https://github.com/YoukoSaint/OptoSync) — 多设备同步采集系统
- 核心参考文件：`core/logging_config.py`、`core/sync_controller.py`、`core/health_monitor.py`、`drivers/keithley_driver.py`

## 许可

MIT

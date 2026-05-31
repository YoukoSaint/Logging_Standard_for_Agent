# 面向运维的 Agent 日志编程规范

> **来源**: OptoSync 项目日志实践萃取  
> **目的**: 引导 AI Agent 写出面向生产运维的可维护、可调试的代码  
> **受众**: AI 编码 Agent（Claude、GPT 等）

---

## 1. 核心原则

### 1.1 日志哲学

**永远为运维人员写日志，而不是为开发者自己**

- 日志是生产环境故障诊断的首要接口
- 每一行日志都必须能回答："发生了什么？何时发生？为什么重要？"
- 假设读者在应急响应时 **无权访问** 源代码

### 1.2 写日志前的三问

每写一行日志之前，先回答：

1. **谁需要它？**（运维人员、开发者、审计员）
2. **何时会用到？**（实时监控、事后分析、合规审计）
3. **能据此采取什么行动？**（若无行动可做，就不要打这条日志）

---

## 2. 日志级别（严格定义）

### 2.1 级别体系

```
DEBUG < INFO < WARNING < ERROR < CRITICAL
```

### 2.2 各级别使用规则

#### **DEBUG** — 开发期诊断信息

**使用场景：**
- 函数进出及参数
- 中间计算结果
- 协议层细节（SCPI 指令、SQL 查询、API 请求）

**OptoSync 示例：**
```python
logger.debug("Trying connection to %s", addr or "auto-discover")
logger.debug("Timing estimates (coarse): spec=%.0fms, keithley=%.0fms", spec_est, keithley_est)
```

**规则：**
- 绝不在 DEBUG 级别打印敏感数据（密码、令牌、个人身份信息）
- 使用 `%s` 格式化，**不用** f-string（延迟求值）
- 生产环境默认关闭

---

#### **INFO** — 正常运行事件

**使用场景：**
- 系统状态变更（启动、停止、连接、断开）
- 配置变更
- 重要操作的完成
- 性能指标（吞吐量、延迟、资源消耗）

**OptoSync 示例：**
```python
logger.info("Connected to %s at %s", idn, addr)
logger.info("Acquisition started (target=%.1f Hz, period=%.1f ms, max=%.1f Hz)",
            target_rate, period_ms, max_rate)
logger.info("CSV logging started: %s, %s", timeseries_path, spectrum_path)
```

**规则：**
- 必须包含量化指标（频率、数量、耗时）
- 状态描述用现在时（"采集已启动"，而非"正在启动采集"）
- 长运行操作必须同时记录 START 和 STOP

---

#### **WARNING** — 可恢复的异常，需要关注

**使用场景：**
- 性能降级（回退到慢速路径）
- 重试行为
- 资源压力（队列积压、内存增长）
- 不阻塞运行的配置问题

**OptoSync 示例：**
```python
logger.warning("Acquisition already running.")
logger.warning("Keithley busy (attempt %d/3), retrying ...", attempt + 1)
logger.warning("Acquisition thread did not stop cleanly.")
```

**规则：**
- 必须包含上下文：什么失败了、为什么、正在如何处理
- 重试必须写明次数
- 每次事件只记录一次，不要在每次重试时重复

---

#### **ERROR** — 操作失败，但系统继续运行

**使用场景：**
- 被重试或跳过的操作失败
- 数据丢失或损坏
- 外部服务故障
- 意外异常（含堆栈跟踪）

**OptoSync 示例：**
```python
logger.error("Device %s measurement error: %s", dev.name, e)
logger.error("Callback error: %s", e)
logger.exception("Error closing %s CSV file", fname)  # 自动包含堆栈
```

**规则：**
- 在 except 块中使用 `logger.exception()`（自动附带 traceback）
- 必须包含异常的报错信息
- 指明哪个组件/设备出故障
- 记录影响范围："丢弃 X 个采样点"、"断开 Y 个客户端"

---

#### **CRITICAL** — 系统故障，需立即处理

**使用场景：**
- 不可恢复的错误，需重启
- 数据完整性被破坏
- 安全入侵
- 资源耗尽（OOM、磁盘满）

**示例模式：**
```python
logger.critical("Out of memory: RSS=%d MB exceeds limit %d MB. Shutting down.", rss, limit)
logger.critical("Database corruption detected in %s. Manual recovery required.", db_path)
```

**规则：**
- 日志后必须执行 shutdown 或安全模式
- 消息中必须包含恢复建议
- 需要立即通知值班工程师

---

## 3. 结构化日志模式

### 3.1 按时间戳切分的运行日志文件

**OptoSync 模式：**
```python
ts = datetime.now().strftime("%Y%m%d_%H%M%S")
log_filepath = os.path.join(log_dir, f"{app_name}_{ts}.log")
```

**为什么这样做：**
- 每次运行的日志文件相互隔离
- 避免了日志轮转的复杂性
- 日志文件名中的时间戳便于与数据文件关联
- 便于事后用 grep 全量检索

**Agent 规则：** 批处理任务、守护进程、长运行进程必须使用时间戳日志文件。

---

### 3.2 双 Handler 配置（文件 + 控制台）

**OptoSync 模式：**
```python
# 文件 Handler：DEBUG 级别，包含完整上下文
file_handler.setLevel(logging.DEBUG)
file_handler.setFormatter(logging.Formatter(
    "%(asctime)s [%(levelname)-7s] %(name)s:%(lineno)d - %(message)s"
))

# 控制台 Handler：INFO 级别，简洁输出
console_handler.setLevel(logging.INFO)
console_handler.setFormatter(logging.Formatter(
    "%(asctime)s [%(levelname)-7s] %(message)s"
))
```

**为什么这样做：**
- 运维人员在终端看到干净的 INFO+ 信息
- 完整的 DEBUG 跟踪保存在文件中用于深度调试
- 文件日志包含模块名 + 行号，方便代码定位

**Agent 规则：** 必须同时配置两种 handler，并使用不同级别和格式。

---

### 3.3 专用协议 Logger

**OptoSync 模式：**
```python
# SCPI 协议流量独立记录（仅在设置 SYNC_DEBUG=1 时启用）
scpi_logger = logging.getLogger("scpi")
scpi_handler = RotatingFileHandler(f"{app_name}_{ts}_scpi.log")
scpi_logger.addHandler(scpi_handler)
```

**为什么这样做：**
- 协议 dump 内容冗长且高度专业化
- 独立文件避免污染主日志
- 可独立控制开关

**Agent 规则：** 以下场景应使用命名子 Logger：
- 协议流量（SQL、HTTP、SCPI、MQTT）
- 性能剖析
- 审计跟踪

---

## 4. 运维日志模式

### 4.1 状态转换

**模式：**
```python
# START — 附带配置参数
logger.info("Acquisition started (target=%.1f Hz, period=%.1f ms, max=%.1f Hz)",
            target_rate, period_ms, max_rate)

# STOP — 附带汇总统计
logger.info("Acquisition stopped: %d points in %.1f s (target=%.1f Hz, actual=%.1f Hz, dropped=%d)",
            count, elapsed, target_rate, actual_rate, dropped)
```

**规则：**
- 启动日志必须附带配置参数
- 停止日志必须附带汇总统计
- 必须包含性能指标（吞吐量、错误率）

---

### 4.2 健康监控

**OptoSync 模式：**
```python
log.info("HEALTH | uptime=%ds | RSS=%.1fMB (Δ%.1f) | CPU=%.1f%% | thr=%d | acq=%dsmp @%.1fHz | stream=%s q=%d/%d",
         uptime, rss_mb, rss_delta, cpu_pct, thread_count, sample_count, rate, stream_status, queue_depth, queue_max)
```

**规则：**
- 使用固定前缀（`HEALTH |`）便于 grep
- 一行包含所有关键指标
- 定期记录（长运行进程 30-60 秒一次）
- 用增量（Δ）展示趋势变化

---

### 4.3 自适应行为

**OptoSync 模式：**
```python
logger.info("Adaptive rate control: actual tick %.0f ms > target %.0f ms. Settling at %.1f Hz.",
            actual_ms, target_ms, adapted_hz)
```

**规则：**
- 系统自动调整参数时必须记录
- 解释原因（实际 > 目标）
- 展示调整结果（新速率）
- 使用 `Adaptive`、`Fallback`、`Throttling` 等关键词便于搜索

---

### 4.4 资源清理

**OptoSync 模式：**
```python
try:
    file.close()
except Exception:
    logger.exception("Error closing %s CSV file", fname)
```

**规则：**
- 清理失败必须记录（它往往预示着资源泄漏）
- 使用 `logger.exception()` 获取堆栈
- 必须包含资源标识（文件名、连接 ID）

---

## 5. 性能友好的日志

### 5.1 热路径日志控制

**反模式：**
```python
# 错误：每条采样都打日志（每秒 10,000 次）
for sample in stream:
    logger.debug("Processing sample %d", sample.id)
```

**正确模式：**
```python
# 正确：每隔 N 条或只在出错时记录
if sample_count % 1000 == 0:
    logger.debug("Processed %d samples", sample_count)
```

**规则：**
- 严禁在高频循环（>100 Hz）内打日志
- 高频事件使用取模采样
- DEBUG 日志移到热路径之外

---

### 5.2 延迟格式化

**反模式：**
```python
# 错误：f-string 在 DEBUG 关闭时也会求值
logger.debug(f"Data: {expensive_repr(obj)}")
```

**正确模式：**
```python
# 正确：% 格式化推迟到需要时才求值
logger.debug("Data: %s", expensive_repr(obj))
```

**规则：**
- 使用 `%s` 格式化，不要用 f-string
- Logger 仅在级别启用时才求值参数
- 生产环境（DEBUG 关闭）可节省大量 CPU

---

## 6. 错误处理与日志

### 6.1 异常日志

**OptoSync 模式：**
```python
try:
    result = device.measure()
except DeviceError as e:
    logger.error("Device %s measurement error: %s", device.name, e)
    dropped_count += 1
except Exception:
    logger.exception("Unexpected error in measurement loop")
    raise
```

**规则：**
- 先捕获具体异常
- 意外异常用 `logger.exception()`（自动包含 traceback）
- 记录影响（丢弃的样本数、失败的请求数）
- 不可恢复时重新抛出

---

### 6.2 重试日志

**OptoSync 模式：**
```python
for attempt in range(3):
    try:
        return operation()
    except TransientError as e:
        if attempt < 2:
            logger.warning("Operation failed (attempt %d/3), retrying: %s", attempt + 1, e)
            time.sleep(backoff)
        else:
            logger.error("Operation failed after 3 attempts: %s", e)
            raise
```

**规则：**
- 重试时打 WARNING
- 最终失败时打 ERROR
- 使用 N/M 格式标明次数
- 不要每次重试都记录（配合指数退避使用）

---

## 7. 上下文与关联追踪

### 7.1 请求 / 会话 ID

**模式：**
```python
logger = logging.getLogger(__name__)

def process_request(request_id, data):
    logger.info("[%s] Processing request: %d bytes", request_id, len(data))
    try:
        result = compute(data)
        logger.info("[%s] Request completed: %d results", request_id, len(result))
    except Exception:
        logger.exception("[%s] Request failed", request_id)
```

**规则：**
- 所有日志加 `[request_id]` 或 `[session_id]` 前缀
- 通过 grep 即可追踪完整请求链路
- 多线程/异步系统不可或缺

---

### 7.2 组件标识

**OptoSync 模式：**
```python
logger = logging.getLogger(__name__)  # 例如 "drivers.keithley_driver"

logger.info("Connected to %s at %s", idn, addr)
# 输出: 2026-05-31 17:45:23 [INFO   ] drivers.keithley_driver:95 - Connected to ...
```

**规则：**
- 使用 `logging.getLogger(__name__)` 自动获取模块名
- 文件 handler 格式中包含行号
- 调试时可以直接根据行号跳转到代码

---

## 8. 指标与可观测性

### 8.1 量化日志

**OptoSync 模式：**
```python
logger.info("Acquisition stopped: %d points in %.1f s (target=%.1f Hz, actual=%.1f Hz, dropped=%d)",
            count, elapsed, target_rate, actual_rate, dropped)
```

**规则：**
- 始终带单位（Hz、MB、ms、%）
- 同时记录目标值和实际值以便对比
- 必须包含错误/丢弃计数
- 精度保持一致（频率用 %.1f，延迟用 %.3f）

---

### 8.2 周期性指标

**OptoSync 模式：**
```python
# 每 30 秒输出一次
log.info("HEALTH | uptime=%ds | RSS=%.1fMB (Δ%.1f) | CPU=%.1f%% | ...")
```

**规则：**
- 使用固定间隔（30-60 秒）
- 用增量（Δ）展示趋势
- 使用统一前缀方便日志解析
- 量大时拆分到独立指标文件

---

## 9. 安全与合规

### 9.1 严禁记录敏感数据

**严禁行为：**
```python
# 绝对不要这样做
logger.debug("API key: %s", api_key)
logger.info("User password: %s", password)
logger.debug("Credit card: %s", cc_number)
```

**正确做法：**
```python
logger.debug("API key: %s", api_key[:8] + "..." if api_key else None)
logger.info("User authenticated: %s", username)
logger.debug("Payment processed: last4=%s", cc_last4)
```

**规则：**
- 脱敏：密码、令牌、密钥、个人身份信息
- 只记录非敏感的标识符（username、user_id、后 4 位）
- 审计日志中用 `[REDACTED]` 占位

---

### 9.2 审计日志

**模式：**
```python
audit_logger = logging.getLogger("audit")
audit_logger.info("USER_ACTION | user=%s | action=%s | resource=%s | result=%s",
                  user_id, action, resource_id, "SUCCESS" if ok else "DENIED")
```

**规则：**
- 使用独立 logger 记录审计跟踪
- 必须包含：谁、做了什么、在什么资源上、结果
- 生产环境永不关闭审计日志
- 使用不可变的追加写文件

---

## 10. Agent 实现检查清单

Agent 在编写代码时，必须检查：

- [ ] 在 `main()` 或模块入口处配置日志
- [ ] 批处理/守护进程使用时间戳日志文件
- [ ] 配置双 handler（文件=DEBUG，控制台=INFO）
- [ ] 每个模块使用 `logging.getLogger(__name__)`
- [ ] 长运行操作记录 START/STOP，附带指标
- [ ] except 块中使用 `logger.exception()`
- [ ] 所有量化日志必须带单位
- [ ] 避免在热路径（>100 Hz 循环）中打日志
- [ ] 使用 `%s` 格式化，不用 f-string
- [ ] 绝不记录密码、令牌、个人身份信息
- [ ] 守护进程添加健康监控（内存、CPU、吞吐量）
- [ ] 使用统一前缀便于 grep（HEALTH、ALERT、AUDIT）

---

## 11. 完整示例：生产级日志配置

```python
"""
示例：Agent 编写的生产级日志配置。
"""
import logging
import logging.handlers
import os
from datetime import datetime

def setup_logging(app_name="MyApp", log_dir="logs", console_level=logging.INFO):
    """配置生产级日志，使用时间戳文件。"""
    os.makedirs(log_dir, exist_ok=True)

    # 时间戳日志文件
    ts = datetime.now().strftime("%Y%m%d_%H%M%S")
    log_path = os.path.join(log_dir, f"{app_name}_{ts}.log")

    # 根 Logger
    root = logging.getLogger()
    root.setLevel(logging.DEBUG)
    root.handlers.clear()

    # 文件 Handler：DEBUG 级别，完整上下文
    file_handler = logging.handlers.RotatingFileHandler(
        log_path,
        maxBytes=10 * 1024 * 1024,  # 10 MB
        backupCount=7,
        encoding="utf-8",
    )
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(logging.Formatter(
        "%(asctime)s [%(levelname)-7s] %(name)s:%(lineno)d - %(message)s"
    ))
    root.addHandler(file_handler)

    # 控制台 Handler：INFO 级别，简洁输出
    console_handler = logging.StreamHandler()
    console_handler.setLevel(console_level)
    console_handler.setFormatter(logging.Formatter(
        "%(asctime)s [%(levelname)-7s] %(message)s"
    ))
    root.addHandler(console_handler)

    logging.info("日志系统初始化完成: %s", log_path)
    return log_path


# ── 模块中的用法 ──────────────────────────────────
logger = logging.getLogger(__name__)

def process_data(data_id: str, payload: bytes) -> list:
    """处理数据并记录完整链路。"""
    logger.info("[%s] 开始处理: %d 字节", data_id, len(payload))
    try:
        result = expensive_operation(payload)
        logger.info("[%s] 处理完成: %d 条结果, 耗时 %.2f 秒",
                    data_id, len(result), elapsed)
        return result
    except ValueError as e:
        logger.error("[%s] 数据校验失败: %s", data_id, e)
        raise
    except Exception:
        logger.exception("[%s] 未知错误", data_id)
        raise
```

---

## 12. 常见反模式

### ❌ 用 print 替代 logging
```python
print("Starting process...")                      # 错误
logger.info("Starting process...")                # 正确
```

### ❌ 无上下文的日志
```python
logger.error("Failed")                            # 错误：什么失败了？
logger.error("设备 %s 连接失败: %s", device_id, e)  # 正确
```

### ❌ 在紧密循环中打日志
```python
for i in range(10000):
    logger.debug("Processing %d", i)              # 错误：10000 行日志
```

### ❌ 日志调用中使用 f-string
```python
logger.debug(f"Value: {expensive_call()}")         # 错误：总是求值
logger.debug("Value: %s", expensive_call())        # 正确：延迟求值
```

### ❌ 静默吞掉异常
```python
try:
    risky_operation()
except Exception:
    pass                                          # 错误：静默失败
```

### ❌ 记录敏感数据
```python
logger.info("Password: %s", password)             # 错误：安全违规
logger.info("用户已认证: %s", username)             # 正确
```

---

## 13. 总结：十条黄金法则

1. **为运维人员写日志，而非开发者自己** —— 假设读者看不到源码
2. **使用结构化格式** —— 时间戳文件名、统一前缀
3. **包含完整上下文** —— 谁、什么、何时、为什么、多少
4. **尊重性能底线** —— 避开热路径、使用延迟格式化
5. **绝不记录密钥** —— 脱敏密码、令牌、个人身份信息
6. **记录状态变更** —— START/STOP 附带指标
7. **合理使用日志级别** —— DEBUG / INFO / WARNING / ERROR / CRITICAL
8. **确保可追踪** —— 请求 ID、模块名、行号
9. **监控系统健康** —— 长运行进程必须有周期性指标
10. **测试你的日志** —— grep 常用模式、验证可读性

---

## 附录：各级别决策速查表

| 场景 | 级别 | 示例消息 |
|------|------|----------|
| 变量值、函数进出、协议细节 | DEBUG | `"Query returned %d rows in %.2f ms"` |
| 服务启动/停止、配置变更、操作完成 | INFO | `"Server listening on :8080 (workers=4)"` |
| 自动降级、重试、资源接近上限 | WARNING | `"Redis unreachable, using local cache (attempt 2/3)"` |
| 请求失败、数据损坏、外部服务不通 | ERROR | `"Payment gateway timeout after 30s: order_id=xxx"` |
| OOM、磁盘满、安全入侵、数据不可逆损坏 | CRITICAL | `"Disk 98% full on /data, shutdown imminent"` |

---

## 附录：日志格式字段说明

| 格式字段 | 含义 | 运维价值 |
|----------|------|----------|
| `%(asctime)s` | 时间戳 | 精确到毫秒的时间线还原 |
| `%(levelname)-7s` | 日志级别（左对齐 7 字符） | 快速过滤 ERROR |
| `%(name)s` | Logger 名称（模块路径） | 快速定位故障模块 |
| `%(lineno)d` | 代码行号 | 直接从日志跳转到源码 |
| `%(message)s` | 日志正文 | 描述事件的自由文本 |

---

**Agent 日志编程规范 全文完**

*基于 OptoSync 项目分析提取（2026-05-31）*

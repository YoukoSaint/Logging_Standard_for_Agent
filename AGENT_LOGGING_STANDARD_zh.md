<!--
🚨 AGENT INSTRUCTION: This is the COMPLETE specification. Read ALL sections before writing any code.
The README.md checklist is a SUMMARY — this file contains the actual implementation requirements.
Key sections to read FIRST:
  - §2 日志级别 (strict DEBUG/INFO/WARNING/ERROR/CRITICAL definitions)
  - §3 setup_logging() pattern (dual Handler, timestamped files)
  - §4 结构化日志模式 (START/STOP pairs, quantified metrics)
  - §5 错误处理 (logger.exception, stack traces)
  - §7 安全合规 (never log secrets/PII)
  - §8 反模式 (what NOT to do)
-->

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

> 完整规范见 **§16 HEALTH & METRICS**（含 GPU/显存、磁盘、FDS、网络、协程、cgroup、阈值表、采集实现模板）。

**OptoSync 模式（最小版）：**
```python
log.info("HEALTH | uptime=%ds | RSS=%.1fMB (Δ%.1f) | CPU=%.1f%% | thr=%d | acq=%dsmp @%.1fHz | stream=%s q=%d/%d",
         uptime, rss_mb, rss_delta, cpu_pct, thread_count, sample_count, rate, stream_status, queue_depth, queue_max)
```

**规则：**
- 使用固定前缀（`HEALTH |`）便于 grep
- 一行包含所有关键指标
- 定期记录（长运行进程 30-60 秒一次；短任务不发 HEALTH，见 §16.7）
- 用增量（Δ）展示趋势变化
- 涉及 GPU/ML 场景必须扩展 `gpu=idx:util|mem|temp|pwr` 字段（§16.2）

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

> 多用户系统必须包含 `user_id` 上下文；完整规范见 **§15 USER_ACTION**。

**模式：**
```python
logger = logging.getLogger(__name__)

def process_request(user_id: str, session_id: str, request_id: str, data):
    logger.info("[user=%s][session=%s][req=%s] Processing request: %d bytes",
                user_id, session_id, request_id, len(data))
    try:
        result = compute(data)
        logger.info("[user=%s][session=%s][req=%s] Request completed: %d results",
                    user_id, session_id, request_id, len(result))
    except PermissionError:
        logger.error("[user=%s][session=%s][req=%s] Permission denied for resource=%s",
                     user_id, session_id, request_id, data.resource_id)
        raise
    except Exception:
        logger.exception("[user=%s][session=%s][req=%s] Request failed",
                         user_id, session_id, request_id)
```

**规则：**
- 所有用户触发的日志加 `[user=<uid>][session=<sid>][req=<rid>]` 三元前缀
- 系统内部事件（cron / 后台任务）使用 `[user=system][session=-][req=<rid>]`
- `user_id` 必须是**内部不可逆 ID**（禁止直接写邮箱 / 手机号 / 真实姓名）
- 通过 grep 即可追踪：(1) 单个用户的所有行为；(2) 单个请求的完整链路；(3) 单个会话的所有交互
- 多线程/异步系统中三个 ID 必须通过 `contextvars` 传递，禁止显式传参
- 当三者任一不可用时，禁止静默省略，必须用占位符 `[user=unknown]` 等

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

> 完整规范见 **§15 USER_ACTION**（字段字典 / 登录生命周期 / 配置变更 / 取消 / 校验失败 / 权限拒绝 / 数据敏感操作）。

**模式（最小版）：**
```python
audit_logger = logging.getLogger("audit")
audit_logger.info("USER_ACTION | user=%s | action=%s | resource=%s | result=%s",
                  user_id, action, resource_id, "SUCCESS" if ok else "DENIED")
```

**规则：**
- 使用独立 logger 记录审计跟踪（推荐 `audit.user` / `audit.auth` / `audit.config` / `audit.permission` 子 logger）
- 必须包含：谁、做了什么、在什么资源上、结果
- 生产环境永不关闭审计日志
- 使用不可变的追加写文件
- 用户字段使用**内部不可逆 ID**（不是邮箱、手机号、真实姓名）
- 敏感值（旧/新配置、用户输入）一律写 `[REDACTED]`

---

## 10. Agent 实现检查清单

Agent 在编写代码时，必须检查：

### 基础日志配置（§3）

- [ ] 在 `main()` 或模块入口处配置日志
- [ ] 批处理/守护进程使用时间戳日志文件
- [ ] 配置双 handler（文件=DEBUG，控制台=INFO）
- [ ] 每个模块使用 `logging.getLogger(__name__)`
- [ ] 使用统一前缀便于 grep（`HEALTH |`、`ALERT |`、`USER_ACTION |`、`FILE_IO |`、`[transport=xxx][addr=xxx]`）

### 通用日志内容（§4 / §6 / §8）

- [ ] 长运行操作记录 START/STOP，附带指标
- [ ] except 块中使用 `logger.exception()`
- [ ] 所有量化日志必须带单位（Hz、MB、ms、%、°C、W）

### 性能与安全（§5 / §9）

- [ ] 避免在热路径（>100 Hz 循环）中打日志
- [ ] 使用 `%s` 格式化，不用 f-string
- [ ] 绝不记录密码、令牌、个人身份信息

### 用户交互（§15）

- [ ] 用户触发的所有事件含 `user_id`（不可逆 ID）
- [ ] 登录/登出/认证失败在 `audit.user` 子 logger 独立记录
- [ ] 配置变更记录 `old=` → `new=`（敏感字段写 `[REDACTED]`）
- [ ] 用户取消/中止（INFO） vs 系统崩溃（CRITICAL）严格区分
- [ ] 输入校验失败记录字段名 + 失败原因，值用 `[REDACTED]`
- [ ] 权限拒绝事件含 `user_id` + `required_role` + `resource`
- [ ] 数据导出/删除/下载记录 `rows=` / `format=` / `confirmation=2FA`

### 系统资源与周期性指标（§16）

- [ ] 守护进程添加健康监控（HEALTH 行）
- [ ] HEALTH 行必含 `uptime / rss / cpu / thr / q` 五项
- [ ] 涉及 GPU 时记录 `gpu=idx:util|mem|temp|pwr`
- [ ] 高并发服务 HEALTH 含 `fds` + `net:est|tw|cw`
- [ ] 写盘型守护进程 HEALTH 含 `disk:mount=pct%`
- [ ] 启动 ≥ 5s 后输出首条 HEALTH；关闭前输出 `final=true` 末条
- [ ] 阈值告警按 §16.4 表触发（WARNING/ERROR/CRITICAL）

### 物理接口（§14 / §17）

- [ ] 每条仪器通信日志都带有 `[transport=xxx][addr=xxx]` 标签
- [ ] 连接/断开在 INFO 级别记录，包含地址和设备身份
- [ ] 意外断开在 WARNING 或 ERROR 级别记录
- [ ] SCPI 协议和传输层使用独立的子 Logger
- [ ] 嵌入式/工业/视觉接口（§17）使用对应 transport 标签
- [ ] 高频物理接口（>100 Hz）用子 Logger + 周期汇总，禁止逐条
- [ ] 启动时记录总线/设备枚举结果（含负结果）

### 文件 I/O（§18）

- [ ] 文件 I/O 日志以 `FILE_IO | op=<op> kind=<kind> path=<path>` 开头
- [ ] 业务数据写入走 `tmp + os.replace` 原子写
- [ ] `fsync` 失败升 CRITICAL
- [ ] 写循环用 `rows % BATCH == 0` 取模聚合日志
- [ ] 大文件按 ≤ 64 MiB 分块
- [ ] 配置加载失败 = CRITICAL
- [ ] 临时文件异常残留清理有 ERROR 记录

> 完整维度化清单见各章节末尾的 §15.8 / §16.9 / §17.9 / §18.9 检查清单。

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

### ❌ 用户操作无 user_id 上下文（违反 §15）
```python
logger.error("Permission denied")                                          # 错误：缺 user/resource 上下文
logger.error("AUTHZ_DENIED | user=%s | resource=%s | required_role=%s",    # 正确
             user_id, resource_id, required_role)
```

### ❌ 高频 I/O 逐条日志（违反 §5.1 / §17.4）
```python
for msg in can_bus:                                                       # 1kHz → 日志爆
    logger.info("[transport=can] recv 0x%03X", msg.arbitration_id)
# 正确：见 §17.4 子 Logger + 周期汇总 + 异常逐条
```

### ❌ 一次性 `f.read()` 加载 GB 级文件（违反 §18.6）
```python
data = open("big.bin", "rb").read()    # OOM
# 正确：按 ≤ 64 MiB 分块
```

### ❌ 直接 `open(..., 'w')` 覆盖生产数据（违反 §18.4）
```python
open("/data/run.csv", "w").write(...)   # 半写即损坏
# 正确：tmp + os.replace 原子写
```

### ❌ `try/except: pass` 吞掉 `fsync` 失败（违反 §18.3）
```python
try: os.fsync(fd)
except: pass                           # 断电即丢数据
# 正确：CRITICAL 级别 + 注明 durability=NOT_GUARANTEED
```

### ❌ GPU/显存完全无监控（违反 §16）
```python
log.info("HEALTH | rss=%.1fMB cpu=%.1f%%", rss, cpu)    # ML 训练静默 OOM
# 正确：含 gpu=idx:util|mem|temp|pwr（见 §16.2）

---

## 13. 总结：十一条黄金法则

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
11. **按维度打标签** —— 每一行日志都能被分桶检索：`USER_ACTION |`、`HEALTH |`、`FILE_IO |`、`[transport=xxx][addr=xxx]`（见 §15-§18）

---

## 附录：各级别决策速查表

| 场景 | 级别 | 示例消息 |
|------|------|----------|
| 变量值、函数进出、协议细节 | DEBUG | `"Query returned %d rows in %.2f ms"` |
| 服务启动/停止、配置变更、操作完成 | INFO | `"Server listening on :8080 (workers=4)"` |
| 自动降级、重试、资源接近上限 | WARNING | `"Redis unreachable, using local cache (attempt 2/3)"` |
| 请求失败、数据损坏、外部服务不通 | ERROR | `"Payment gateway timeout after 30s: order_id=xxx"` |
| OOM、磁盘满、安全入侵、数据不可逆损坏 | CRITICAL | `"Disk 98% full on /data, shutdown imminent"` |
| **用户登录成功 / 主动登出 / 合法配置** | INFO | `"USER_ACTION \| user=alice \| action=LOGIN \| result=SUCCESS"` |
| **登录失败（密码错）/ 校验失败** | WARNING | `"USER_ACTION \| user=alice \| action=LOGIN \| result=FAIL \| reason=invalid_password"` |
| **权限被拒绝 / 触发未捕获异常** | ERROR | `"USER_ACTION \| user=bob \| action=DELETE \| result=DENIED \| reason=insufficient_role"` |
| **账号被盗用 / 批量权限提升尝试** | CRITICAL | `"USER_ACTION \| action=PRIV_ESCALATION \| result=DENIED \| attempts=15"` |
| **GPU 显存 / 磁盘使用率越界** | WARNING/CRITICAL | `"HEALTH \| gpu=0:mem=24000/24576MB(98%) \| disk:data=96%"` |
| **文件 I/O 失败**（按 §18.7 表） | ERROR/CRITICAL | `"FILE_IO \| op=fsync kind=data path=/x FAILED durability=NOT_GUARANTEED"` |
| **物理接口异常断开 / NACK / Bus-Off** | WARNING/ERROR | `"[transport=can][addr=can0@1Mbps] Bus-Off: TEC=255 REC=0"` |

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



---

## 14. 仪器自动化传输层日志

> 针对多设备自动化系统，仪器通过异构传输通道（USB、Ethernet/IP、Serial、GPIB）通信。
> 每一行日志都必须标明设备是**通过什么方式**通信的，因为故障模式和排查路径因传输层而异。

### 14.1 传输层标记（强制要求）

凡是涉及仪器通信的日志行，**必须**包含传输层标签：

```python
logger.info("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] 已连接")
logger.error("[transport=tcp][addr=192.168.1.100:5025] 命令超时，5s 无响应")
logger.warning("[transport=serial][addr=COM3@115200] 缓冲区溢出，正在清空")
```

**标签格式：**

| 传输层 | 标签 | 地址格式 |
|--------|------|----------|
| USB (USBTMC/VISA) | `transport=usb` | `USB0::VID::PID::serial::INSTR` |
| TCP/IP (VXI-11, Raw Socket) | `transport=tcp` | `host:port` |
| 串口 / RS-232 | `transport=serial` | `port@baud` |
| GPIB | `transport=gpib` | `GPIB0::pad` |
| 未知 / 自动发现 | `transport=auto` | 包含发现方式 |

**规则：**
- 始终使用 `[transport=xxx][addr=yyy]` 作为日志级别之后的头两个字段
- 地址格式在全代码库中保持一致，确保 `grep addr=` 能跨模块检索
- 当一台仪器可通过多条路径访问（如 USB 和 TCP 皆可），标记当前活跃的那条

---

### 14.2 连接生命周期

每次连接、断开和重连**必须**在 **INFO** 级别记录：

```python
# 连接
logger.info("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
            "已连接: idn='Keithley 2450, 4430138, v3.0'")

# 正常断开
logger.info("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
            "已断开: 用户主动关闭")

# 意外断开（USB 热插拔 / 线缆脱落）
logger.warning("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
               "连接丢失: 设备从总线移除")

# 自动重连
logger.info("[transport=tcp][addr=192.168.1.100:5025] "
            "重连成功: 离线 12.3s (共尝试 3 次)")

# 重连失败
logger.error("[transport=tcp][addr=192.168.1.100:5025] "
             "重连失败，已尝试 5 次 (最后一次错误: 连接被拒绝)")
```

**连接健康事件（WARNING 级别）：**

```python
# USB 总线复位
logger.warning("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
               "检测到 USB 总线复位，重新获取句柄...")

# TCP 保活失败
logger.warning("[transport=tcp][addr=192.168.1.100:5025] "
               "TCP 保活超时，正在探测连接...")

# 串口帧错误
logger.warning("[transport=serial][addr=COM3@115200] "
               "串口帧错误 (byte 0x%02x)，正在重新同步...")
```

---

### 14.3 健康监控中的传输层指标

在周期性 HEALTH 行（第 4.2 节）中扩展每类传输层的指标：

```python
log.info("HEALTH | uptime=%ds | "
         "tcp:lat=%.1fms|retx=%d|pool=%d/%d | "
         "usb:tx=%dKB/s|err=%d | "
         "serial:buf=%d/%d|break=%d",
         uptime,
         tcp_latency_ms, tcp_retx_count, tcp_pool_used, tcp_pool_max,
         usb_tx_kbps, usb_err_count,
         serial_buf_used, serial_buf_size, serial_break_count)
```

**各类传输层的关键指标：**

| 传输层 | 指标 |
|--------|------|
| TCP | 延迟(ms)、重传次数、连接池使用量、Socket 超时次数 |
| USB | 吞吐量(KB/s)、传输错误次数、总线复位次数 |
| 串口 | 缓冲区水位、帧错误次数、break 信号次数 |
| GPIB | 超时次数、EOS 错误次数、总线竞争次数 |

**规则：**
- 零错误为正常状态——记录在 DEBUG；周期内首次出错升至 WARNING
- 追踪自上次 HEALTH 行以来的**增量**（非累计值）
- 即使指标为零，每行 HEALTH 也须包含传输层指标

---

### 14.4 协议层与传输层分离

将 SCPI（或其他仪器协议）的流量记录在独立的**子 Logger** 中，与传输层日志分离：

```python
# Logger 层级:
#   "controller"           -> 业务逻辑（主日志）
#   "controller.scpi"      -> 发往仪器的 SCPI 指令
#   "controller.transport" -> TCP/UART 原始字节、USB bulk 传输

logger = logging.getLogger("controller")
scpi_log = logging.getLogger("controller.scpi")
transport_log = logging.getLogger("controller.transport")
```

**使用示例：**

```python
# 业务层日志
logger.info("[transport=tcp][addr=192.168.1.100:5025] 测量通道 1 电压")

# SCPI 日志（独立文件，通过 SCPI_DEBUG=1 启用）
scpi_log.debug("-> *IDN?")
scpi_log.debug("<- Keithley 2450, 4430138, v3.0")

# 传输层日志（独立文件，通过 TRANS_DEBUG=1 启用）
transport_log.debug("已发送: TCP 流 7 字节")
transport_log.debug("已接收: 41 字节，耗时 2.3ms (1 个 TCP segment)")
```

**好处：**
- 主日志保持干净——协议噪音不会掩盖应用事件
- SCPI 日志可通过 grep 回答"哪条命令导致了仪器报错"
- 传输层日志暴露底层问题（部分读取、TCP 分片）

---

### 14.5 仪器发现日志

自动发现是仪器自动化中最脆弱、最难调试的阶段。必须详实记录：

```python
# 发现开始
logger.info("[discover][usb] 正在扫描 USB 总线查找 USBTMC 设备...")
logger.info("[discover][tcp] 正在探测子网 192.168.1.0/24 端口 5025...")

# 发现结果
logger.info("[discover][usb] 发现 3 台仪器: "
            "USB0::0x05E6::0x2450::4430138::INSTR (Keithley 2450), "
            "USB0::0x1AB1::0x0588::DG4162-12345::INSTR (Rigol DG4162)")

logger.info("[discover][tcp] 发现 2 台仪器: "
            "192.168.1.100:5025 (KEITHLEY 2450), "
            "192.168.1.101:5025 (RIGOL DG4162)")

# 发现失败（最重要）
logger.warning("[discover][tcp] mDNS 查询 'KEITHLEY*' 返回 0 台设备 (超时=5s)")
logger.warning("[discover][usb] USB 总线上未发现 USBTMC 设备 (请检查: 驱动? 供电?)")

# 名称匹配与容错
logger.warning("[discover] 配置要求 'spectrometer'，未找到精确匹配；"
               "最接近的 USB 设备: USB0::0x1AB1::0x0588::DG4162-12345::INSTR")
```

**规则：**
- 无论是否找到设备，扫描每条传输通道都须记录（负结果同样重要）
- 包含完整的仪器识别字符串，便于事后比对
- 记录不匹配和降级逻辑——这是"在我桌上能跑"类 bug 的头号来源

---

### 14.6 传输层错误分类

不同传输层的错误模式不同，应以支持按传输层告警的方式记录：

```python
# TCP 错误
logger.error("[transport=tcp][addr=192.168.1.100:5025] "
             "连接被拒绝 (目标端口关闭或仪器未开机)")
logger.error("[transport=tcp][addr=10.0.0.50:5025] "
             "连接超时 (无法路由到主机? 子网错误?)")
logger.warning("[transport=tcp][addr=192.168.1.100:5025] "
               "部分读取: 期望 1024 字节，实际收到 312 字节 (TCP 分片)")

# USB 错误
logger.error("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
             "USB 传输失败: LIBUSB_ERROR_PIPE (端点 stall)")
logger.error("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
             "USB 传输失败: LIBUSB_ERROR_NO_DEVICE (线缆断开)")
logger.warning("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
               "USB 带宽不足: 请求 512 字节，最大包 64 字节")

# 串口错误
logger.error("[transport=serial][addr=COM3@115200] "
             "串口超时: 5s 内无响应 (波特率错误?)")
logger.warning("[transport=serial][addr=COM3@115200] "
               "串口溢出: 丢失 32 字节 (流控问题?)")

# GPIB 错误
logger.error("[transport=gpib][addr=GPIB0::12] "
             "GPIB 超时: 10s 内未收到 SRQ")
logger.warning("[transport=gpib][addr=GPIB0::12] "
               "GPIB EOS 错误: 意外的终止符 0x%02x")
```

---

### 14.7 传输层 Agent 检查清单

在编写仪器自动化代码时，Agent 必须额外检查：

- [ ] 每条仪器通信日志都带有 `[transport=xxx][addr=xxx]` 标签
- [ ] 连接/断开在 INFO 级别记录，包含地址和设备身份
- [ ] 意外断开在 WARNING 或 ERROR 级别记录
- [ ] HEALTH 行包含每条传输层的指标（延迟、错误、吞吐量）
- [ ] SCPI 协议和传输层使用独立的子 Logger
- [ ] 发现阶段记录每条扫描的传输通道及其结果（包括负结果）
- [ ] 传输层特定错误消息包含可操作的恢复建议
- [ ] 仪器地址在所有日志行中使用规范的统一格式

---

## 15. USER_ACTION 用户交互与审计日志

> 现有 §9.2 仅给出 4 行模板，无法支撑多用户系统、合规审计、配置变更追溯等场景。本节把"用户交互"提升为一级规范。

### 15.1 字段字典（强制）

所有用户交互事件统一以下列字段顺序记录，便于 `grep` / `awk` / 日志解析器处理：

```
USER_ACTION | ts=ISO8601 | user=<uid> | role=<role> | session=<sid> |
            tenant=<tid> | action=<verb> | resource=<id> |
            result=SUCCESS|DENIED|FAIL | reason=<why>
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `user` | ✓ | **内部不可逆 ID**（不是邮箱、手机号、真实姓名） |
| `role` | -   | `admin` / `operator` / `guest` / 自定义角色 |
| `session` | - | 与 §7.1 `req=` 关联的会话 ID |
| `tenant` | - | 多租户场景的租户 ID；单租户可省略 |
| `action` | ✓ | 动作动词（见 §15.2-§15.7 清单） |
| `resource` | - | 被操作的资源 ID |
| `result` | ✓ | `SUCCESS` / `DENIED` / `FAIL` |
| `reason` | - | 失败原因（密码错、权限不足、校验失败） |

### 15.2 登录生命周期

```python
audit = logging.getLogger("audit.user")

# 登录成功
audit.info("USER_ACTION | user=%s | role=%s | session=%s | action=LOGIN | result=SUCCESS | src=%s",
           user_id, role, session_id, source_ip)

# 登录失败（密码错、账号不存在）
audit.warning("USER_ACTION | user=%s | action=LOGIN | result=FAIL | reason=%s | src=%s",
              user_id, "invalid_password", source_ip)

# 账号被锁（连续失败）
audit.critical("USER_ACTION | user=%s | action=LOGIN | result=DENIED | reason=account_locked | attempts=%d",
               user_id, failed_attempts)

# 登出
audit.info("USER_ACTION | user=%s | session=%s | action=LOGOUT | result=SUCCESS", user_id, session_id)

# 会话超时
audit.info("USER_ACTION | user=%s | session=%s | action=SESSION_TIMEOUT | duration=%ds",
           user_id, session_id, session_duration_sec)
```

### 15.3 配置变更审计（旧值→新值）

```python
# 普通配置变更
audit.info("USER_ACTION | user=%s | session=%s | action=CONFIG_SET | resource=%s | "
           "key=%s | old=%s | new=%s | result=SUCCESS",
           user_id, session_id, service_id, key, old_val, new_val)

# 敏感凭据：写入 [REDACTED] 而非明文
audit.info("USER_ACTION | user=%s | action=CONFIG_SET | key=api_token | "
           "old=[REDACTED] | new=[REDACTED] | result=SUCCESS", user_id)
```

### 15.4 用户取消 / 中止

区分"用户主动停止"与"系统崩溃"对运维至关重要：

```python
audit.info("USER_ACTION | user=%s | session=%s | action=CANCEL | resource=%s | "
           "reason=user_requested | progress=%.0f%% | elapsed=%.1fs",
           user_id, session_id, job_id, progress_pct, elapsed_s)
```

### 15.5 输入校验失败

值必须 `[REDACTED]`，否则会泄露 PII：

```python
audit.warning("USER_ACTION | user=%s | action=INPUT_VALIDATE | resource=%s | "
              "field=%s | reason=%s | value=[REDACTED] | result=FAIL",
              user_id, form_id, field_name, validation_error)
```

### 15.6 权限拒绝

```python
audit.error("USER_ACTION | user=%s | role=%s | action=%s | resource=%s | "
            "required_role=%s | result=DENIED | reason=insufficient_role",
            user_id, role, attempted_action, resource_id, required_role)
```

### 15.7 数据敏感操作（导出/删除/下载）

GDPR / SOC2 合规要求：

```python
audit.info("USER_ACTION | user=%s | session=%s | action=EXPORT_DATA | "
           "resource=%s | rows=%d | format=%s | confirmation=2FA | result=SUCCESS",
           user_id, session_id, dataset_id, row_count, export_format)
```

### 15.8 §15 检查清单

- [ ] 用户触发的所有事件含 `user_id`（不可逆 ID，非邮箱/手机号）
- [ ] 登录/登出/认证失败在 `audit.user` 子 logger 独立记录
- [ ] 配置变更记录 `old=` → `new=`（敏感字段写 `[REDACTED]`）
- [ ] 用户取消/中止（INFO） vs 系统崩溃（CRITICAL）日志级别严格区分
- [ ] 输入校验失败记录字段名 + 失败原因，值用 `[REDACTED]`
- [ ] 权限拒绝事件含 `user_id` + `required_role` + `resource`
- [ ] 数据导出/删除/下载记录 `rows=` / `format=` / `confirmation=2FA`
- [ ] `audit` logger 永不关闭（生产环境 DEBUG 模式下也至少 INFO 开启）

---

## 16. HEALTH & METRICS 系统资源与周期性指标

> 现有 §4.2 仅给出一条演示性 HEALTH 行，对 2024+ 的 ML/AI Agent 工程严重不足：GPU/显存 = 0 覆盖；磁盘/FDS/网络/协程/cgroup = 0 覆盖；阈值表、采集库、跨平台策略全部缺失。

### 16.1 必选 vs 推荐指标

**必选（任何长运行进程都必须出现）**：

| 字段 | 含义 | 备注 |
|------|------|------|
| `uptime` | 进程启动时长（秒） | 关闭时记 `final=true` |
| `rss` | 进程 RSS（MB）+ `Δ` 自上次 HEALTH 的变化 | |
| `cpu` | 进程 CPU 占用 % | |
| `thr` | 线程数 | |
| `q` | 业务队列当前/上限 | |

**推荐（按进程类型选用）**：

| 字段 | 适用场景 | 来源 |
|------|---------|------|
| `gpu=<idx>:util=\|mem=\|temp=\|pwr=` | ML 训练/推理 | `pynvml` / `torch.cuda` |
| `disk:<mount>=<pct>%` | 写盘型守护进程 | `psutil.disk_usage` |
| `fds=<used>/<max>` | 高并发服务 | `psutil.Process.num_fds` |
| `net:est=\|tw=\|cw=` | API server / 消息消费者 | `psutil.net_connections` |
| `load=<avg1>` | CPU 调度诊断 | `psutil.getloadavg` |
| `coro=<n>` | asyncio 项目 | `asyncio.all_tasks` |

### 16.2 HEALTH 行标准格式

```python
log.info(
    "HEALTH | "
    "uptime=%ds | "
    "rss=%.1fMB (Δ%+.1f) | "
    "cpu=%.1f%% (load=%.2f) | "
    "thr=%d coro=%d | "
    "fds=%d/%d | "
    "net:est=%d|tw=%d|cw=%d | "
    "gpu=%d:util=%d%%|mem=%d/%dMB(%.0f%%)|temp=%dC|pwr=%dW | "
    "disk:root=%.0f%%|data=%.0f%%|log=%.0f%% | "
    "q=%d/%d | "
    "rate=%.1f/s",
    uptime_sec,
    rss_mb, rss_delta_mb,
    cpu_pct, load_avg_1,
    thread_count, coro_count,
    fds_used, fds_max,
    net_est, net_tw, net_cw,
    gpu_idx, gpu_util, gpu_mem_used_mb, gpu_mem_total_mb, gpu_mem_pct, gpu_temp_c, gpu_power_w,
    disk_root_pct, disk_data_pct, disk_log_pct,
    queue_depth, queue_max,
    msg_rate,
)
```

**多卡场景**：`gpu=0:...|gpu=1:...` 拼接；不要只记首张卡。

### 16.3 推荐采集库

| 库 | 用途 | 平台 |
|----|------|------|
| `psutil` | 跨平台进程/系统指标 | Linux / macOS / Windows |
| `pynvml` | NVIDIA GPU 指标（util/mem/temp/pwr） | NVIDIA 驱动存在时 |
| `torch.cuda` | 训练框架侧的显存分配 | PyTorch 项目 |
| `GPUtil` | pynvml 的高层封装 | 同上 |

macOS 上无 `/proc`；Windows 需 `wmi` / `psutil`。Apple Silicon（MPS）的显存统计需通过 `torch.mps` / Metal API，**目前无统一库**——若 Agent 编写 macOS ML 工具，建议在 README 注明限制。

### 16.4 阈值告警表（默认值，可调整）

| 指标 | WARNING | ERROR | CRITICAL |
|------|---------|-------|----------|
| RSS 增长率 | > 50 MB/h | > 200 MB/h | 单调上升 1h 无回落 |
| 磁盘使用率 | > 80% | > 90% | > 95% 或 inode > 90% |
| 显存使用率 | > 85% | > 95% | = 100%（即将 OOM） |
| GPU 温度 | > 80°C | > 88°C | > 95°C（降频或关机） |
| FDs 使用率 | > 70% ulimit | > 85% ulimit | = ulimit（连接失败） |
| CLOSE_WAIT | > 100 | > 1000 | > 10000（连接泄漏） |
| Load average | > 核数 × 1.5 | > 核数 × 3 | > 核数 × 5 |

### 16.5 采集实现模板（psutil + pynvml）

```python
import logging, time
from typing import Optional

try:
    import psutil
except ImportError:
    psutil = None
try:
    import pynvml
    pynvml.nvmlInit()
    _NVML_OK = True
except Exception:
    _NVML_OK = False

log = logging.getLogger("health")
PROC_START = time.monotonic()

def collect_health(prev_rss_mb: Optional[float] = None) -> dict:
    """采集一次指标。任意子项失败不得抛异常。"""
    out = dict(uptime_sec=int(time.monotonic() - PROC_START),
               rss_mb=0.0, rss_delta_mb=0.0, cpu_pct=0.0, load_avg_1=0.0,
               thread_count=0, coro_count=0, fds_used=0, fds_max=0,
               net_est=0, net_tw=0, net_cw=0,
               disk_root_pct=0.0, disk_data_pct=0.0, disk_log_pct=0.0)
    if psutil is None:
        log.warning("psutil not installed; HEALTH metrics limited")
        return out
    proc = psutil.Process()
    rss = proc.memory_info().rss / 1024 / 1024
    out["rss_mb"] = rss
    if prev_rss_mb is not None:
        out["rss_delta_mb"] = rss - prev_rss_mb
    out["cpu_pct"] = proc.cpu_percent(interval=None)
    out["thread_count"] = proc.num_threads()
    try:
        out["fds_used"] = proc.num_fds()
        out["fds_max"] = psutil.Process().rlimit(psutil.RLIMIT_NOFILE)[1]
    except (AttributeError, OSError):
        pass
    try:
        out["load_avg_1"] = psutil.getloadavg()[0]
    except (AttributeError, OSError):
        pass
    try:
        for c in psutil.net_connections(kind="tcp"):
            s = getattr(c, "status", None)
            if s == psutil.CONN_ESTABLISHED: out["net_est"] += 1
            elif s == psutil.CONN_TIME_WAIT:   out["net_tw"]  += 1
            elif s == psutil.CONN_CLOSE_WAIT:  out["net_cw"]  += 1
    except (psutil.AccessDenied, OSError):
        pass
    for mount, key in [("/", "disk_root_pct"), ("/data", "disk_data_pct"),
                       ("/var/log", "disk_log_pct")]:
        try:
            out[key] = psutil.disk_usage(mount).percent
        except (FileNotFoundError, OSError):
            pass
    if _NVML_OK:
        try:
            gpus = []
            for i in range(pynvml.nvmlDeviceGetCount()):
                h = pynvml.nvmlDeviceGetHandleByIndex(i)
                u = pynvml.nvmlDeviceGetUtilizationRates(h)
                m = pynvml.nvmlDeviceGetMemoryInfo(h)
                t = pynvml.nvmlDeviceGetTemperature(h, pynvml.NVML_TEMPERATURE_GPU)
                p = int(pynvml.nvmlDeviceGetPowerUsage(h) / 1000.0)
                gpus.append(dict(idx=i, util=u.gpu,
                                 mem_used=m.used // (1024*1024),
                                 mem_total=m.total // (1024*1024),
                                 mem_pct=100.0 * m.used / m.total,
                                 temp_c=t, power_w=p))
            out["gpus"] = gpus
        except Exception as e:
            log.warning("NVML collection failed: %s", e)
    return out

def format_health(h: dict) -> str:
    parts = [f"uptime={h['uptime_sec']}s",
             f"rss={h['rss_mb']:.1f}MB (Δ{h['rss_delta_mb']:+.1f})",
             f"cpu={h['cpu_pct']:.1f}% (load={h['load_avg_1']:.2f})",
             f"thr={h['thread_count']} coro={h['coro_count']}",
             f"fds={h['fds_used']}/{h['fds_max']}",
             f"net:est={h['net_est']}|tw={h['net_tw']}|cw={h['net_cw']}",
             f"disk:root={h['disk_root_pct']:.0f}%|"
             f"data={h['disk_data_pct']:.0f}%|log={h['disk_log_pct']:.0f}%"]
    for g in h.get("gpus", []):
        parts.append(f"gpu={g['idx']}:util={g['util_pct']}%|"
                     f"mem={g['mem_used_mb']}/{g['mem_total_mb']}MB({g['mem_pct']:.0f}%)"
                     f"|temp={g['temp_c']}C|pwr={g['power_w']}W")
    return "HEALTH | " + " | ".join(parts)
```

### 16.6 周期性输出循环

```python
HEALTH_INTERVAL = 30  # 秒
_prev = None
while not shutdown.is_set():
    h = collect_health(prev_rss_mb=_prev)
    _prev = h["rss_mb"]
    log.info(format_health(h))
    shutdown.wait(HEALTH_INTERVAL)

# 关闭前最后一条
log.info("HEALTH | final=true | " + format_health(collect_health(_prev)).removeprefix("HEALTH | "))
```

### 16.7 短任务策略

任务运行时间 < 2 × `HEALTH_INTERVAL` 时，**不必发 HEALTH**——改为任务结束 SUMMARY 即可：

```python
# < 60s 的批处理：start log + end summary 即足
log.info("BATCH started: rows=%d input=%s", row_count, input_path)
result = process(input_path)
log.info("BATCH completed: rows=%d dur=%.2fs ok=%d fail=%d",
         row_count, elapsed, ok_count, fail_count)
```

### 16.8 容器 / cgroup 注意事项

- 容器内 `RSS` 可能被 cgroup 限制截断（`memory.limit_in_bytes`）
- K8s 部署应额外采集 `cgroup_mem_used / cgroup_mem_max`，避免运维误判
- `cgroup_cpu_quota` 与 `cpu.shares` 应在启动日志 INFO 记录（决定 CPU % 解读）

### 16.9 §16 检查清单

- [ ] HEALTH 行必含 `uptime / rss / cpu / thr / q` 五项
- [ ] 涉及 GPU 时记录 `gpu=idx:util|mem|temp|pwr`
- [ ] 高并发服务 HEALTH 含 `fds` + `net:est|tw|cw`
- [ ] 写盘型守护进程 HEALTH 含 `disk:mount=pct%`
- [ ] 启动 ≥ 5s 后输出首条 HEALTH；关闭前输出 `final=true` 末条
- [ ] 阈值告警按 §16.4 表触发（WARNING/ERROR/CRITICAL）
- [ ] 使用 `psutil` + `pynvml`（NVIDIA）或 README 注明 macOS/AMD 限制
- [ ] 容器部署额外记录 cgroup 限制值
- [ ] 短任务（< 2×周期）不发 HEALTH，改用 START/STOP SUMMARY

---

## 17. Physical Interface Layer 通用物理接口日志

> §14 仪器自动化传输层是**测量仪器专用**（USB-TMC/VISA/TCP-VXI-11/Serial/GPIB）。本节扩展至广义物理接口：嵌入式 MCU、树莓派、工业 PLC、机器视觉、运动控制。

### 17.1 传输层标签扩展

沿用 §14.1 的 `[transport=xxx][addr=xxx]` 格式，新增以下标签：

| 传输层 | 标签 | 地址格式 | 典型场景 |
|--------|------|----------|----------|
| GPIO | `transport=gpio` | `chip=0/line=17` 或 `PIN17` | 树莓派、MCU、PLC DO |
| I²C | `transport=i2c` | `/dev/i2c-1@0x48` | 传感器、EEPROM、ADC |
| SPI | `transport=spi` | `/dev/spidev0.0@1MHz` | FPGA、Flash、高速 ADC |
| MCU UART | `transport=uart` | `USART2@115200` / `ttyAMA0@9600` | MCU 调试、TTL 串口 |
| CAN / CAN-FD | `transport=can` | `can0@1Mbps` / `can0@500kbps-fd` | 汽车 ECU、工业现场 |
| Modbus RTU/TCP | `transport=modbus` | `slave=0x01@COM3@9600` / `tcp:host:502/slave=0x01` | PLC、变频器、电表 |
| OPC-UA | `transport=opcua` | `opc.tcp://host:4840/ns=2;s=Channel1` | SCADA、工业 4.0 |
| USB 摄像头 (UVC) | `transport=uvc` | `/dev/video0@MJPEG@1920x1080@30fps` | Webcam、机器视觉 |
| CSI 摄像头 | `transport=csi` | `csi0@imx477@4056x3040` | 树莓派相机 |
| GigE Vision | `transport=gige` | `192.168.1.50@GVSP@Basler-acA1920` | 工业相机 |
| 步进 / 伺服脉冲 | `transport=pulse` | `AXIS0@dir=DIR1,pul=PUL2,mode=CW/CCW` | 步进电机、伺服 |
| PWM | `transport=pwm` | `chip=0/channel=0@1kHz@50%` | 舵机、调光 |
| 编码器 | `transport=qenc` | `chip=0/quad=0` | 位置反馈 |
| 蓝牙 BLE | `transport=ble` | `AA:BB:CC:DD:EE:FF@GATT` | 低功耗蓝牙 |
| LoRa | `transport=lora` | `sx1276@868MHz@SF7/BW125` | 远距离低功耗 |
| PCIe DAQ 卡 | `transport=pcie` | `NI-PXI-6366/slot=2/ai0` | 数据采集 |

**地址格式规则**：
- Linux 设备路径优先：`/dev/i2c-1`、`/dev/spidev0.0`、`/dev/video0`、`can0`
- 物理参数显式编码：`@0x48`（I²C 地址）、`@1Mbps`（CAN 速率）、`@1920x1080@30fps`（摄像头分辨率帧率）
- 多人协作项目必须在 README 维护"地址字典"，避免每个模块自创格式

### 17.2 总线枚举与发现

嵌入式启动时**必须枚举**所有用到的总线和设备，**含负结果**：

```python
logger.info("[discover][i2c] 扫描 /dev/i2c-1 查找从设备...")
logger.info("[discover][i2c] 找到 3 个设备: 0x48 (TMP102), 0x68 (MPU6050), 0x77 (BMP280)")
logger.warning("[discover][can] can0 接口未启动 (检查: ip link set can0 up?)")
logger.error("[discover][uvc] /dev/video0 不存在 (摄像头未插入?)")
logger.info("[discover][gpio] GPIO17 已配置为 OUTPUT (默认 LOW)")
logger.warning("[discover][gpio] GPIO4 已被占用 (used by: gpio_led.py)，本次跳过")
```

### 17.3 连接生命周期（沿用 §14.2 模式）

```python
# I²C 设备初始化
logger.info("[transport=i2c][addr=/dev/i2c-1@0x48] 已初始化: chip=MPU6050, who_am_i=0x68")

# CAN 通道建立
logger.info("[transport=can][addr=can0@1Mbps] 已 up: ip link set can0 up type can bitrate 1000000")

# 摄像头打开
logger.info("[transport=uvc][addr=/dev/video0@MJPEG@1920x1080@30fps] 已打开: idn='Logitech BRIO, USB 3.0'")

# 意外断开（CAN bus-off）
logger.warning("[transport=can][addr=can0@1Mbps] 进入 Bus-Off 状态，5s 后尝试恢复...")
```

### 17.4 原子操作日志粒度（关键新增）

物理接口调用往往是**高频、原子化**的，粒度规则与业务日志完全不同：

| 操作频率 | 建议粒度 | 示例 |
|----------|----------|------|
| < 10 Hz | 每条都打 INFO / DEBUG | GPIO LED 切换、继电器控制 |
| 10 - 100 Hz | 启动/停止打 INFO，每 100 条取样 DEBUG | UART 文本命令、Modbus 轮询 |
| 100 - 1000 Hz | 仅错误打 WARNING/ERROR；启动/停止 + HEALTH 周期汇总 | I²C 传感器采样、SPI ADC 扫描 |
| > 1000 Hz | **禁止逐条日志**；用专用子 Logger + 周期计数 + 异常时记 | CAN 1Mbps 总线负载、SPI DMA 突发 |

**反例**（1kHz CAN 总线逐条日志会爆）：

```python
# ❌ 1000 行/秒 → 日志文件爆 → IO 阻塞 → 实际丢帧
for msg in can_bus:
    logger.info("[transport=can][addr=can0@1Mbps] recv 0x%03X: %s",
                msg.arbitration_id, msg.data.hex())
```

**正例**（子 Logger + 周期汇总 + 异常逐条）：

```python
can_log = logging.getLogger("transport.can")
frame_count = err_count = 0
last_summary = time.monotonic()
for msg in can_bus:
    frame_count += 1
    if msg.is_error_frame:
        can_log.warning("[transport=can][addr=can0@1Mbps] 错误帧: id=0x%03X, err=%s",
                        msg.arbitration_id, msg.error)
        err_count += 1
    elif msg.arbitration_id in CRITICAL_IDS:
        can_log.info("[transport=can][addr=can0@1Mbps] 关键帧: id=0x%03X, data=%s",
                     msg.arbitration_id, msg.data.hex())
    if time.monotonic() - last_summary >= 1.0:
        can_log.debug("[transport=can][addr=can0@1Mbps] 1s 汇总: frames=%d, err=%d, busload=%.1f%%",
                      frame_count, err_count, busload_pct)
        frame_count = err_count = 0
        last_summary = time.monotonic()
```

### 17.5 嵌入式总线（GPIO / I²C / SPI / UART）

```python
# GPIO
logger.debug("[transport=gpio][addr=chip=0/line=17] set HIGH (mode=OUTPUT, pull=UP)")
logger.warning("[transport=gpio][addr=chip=0/line=4] 引脚冲突: 已被 gpio_led.py 占用，本次 write 跳过")
logger.error("[transport=gpio][addr=chip=0/line=27] 引脚读取失败: gpiod_line_request_get failed (ENODEV)")

# I²C
logger.debug("[transport=i2c][addr=/dev/i2c-1@0x48] reg_read reg=0x00 len=2 -> 0x%04x", value)
logger.warning("[transport=i2c][addr=/dev/i2c-1@0x48] NACK 重试: 3/3 仍失败, 检查接线/上拉")
logger.error("[transport=i2c][addr=/dev/i2c-1@0x68] 总线错误: bus=-EBUSY (其他 master 占用?)")

# SPI
logger.debug("[transport=spi][addr=/dev/spidev0.0@1MHz] xfer len=%d mode=%d cs_high=%s",
             len(tx), mode, cs_active)
logger.warning("[transport=spi][addr=/dev/spidev0.0@1MHz] 时钟分频: 请求 1MHz, 实际 1.024MHz (硬件约束)")
```

### 17.6 工业 / 汽车总线（CAN / Modbus / OPC-UA）

```python
# CAN
logger.info("[transport=can][addr=can0@1Mbps] send id=0x18FEF100 DLC=8 data=%s", data.hex())
logger.warning("[transport=can][addr=can0@1Mbps] 仲裁失败: id=0x%03X 在 %d 次重发后放弃", can_id, retries)
logger.error("[transport=can][addr=can0@1Mbps] Bus-Off: TEC=%d REC=%d, 进入恢复流程", tec, rec)

# Modbus
logger.debug("[transport=modbus][addr=tcp:192.168.1.10:502/slave=0x01] fc=0x03 addr=40001 len=10 -> %s",
             regs.hex())
logger.warning("[transport=modbus][addr=slave=0x01@COM3@9600] CRC 错误: 期望 0x%04x, 实际 0x%04x (电磁干扰?)",
               expected, actual)
logger.error("[transport=modbus][addr=tcp:192.168.1.10:502/slave=0x01] 从站无响应: fc=0x06 写入 40001, 超时 3s")

# OPC-UA
logger.info("[transport=opcua][addr=opc.tcp://plc.local:4840] 已连接: server='Siemens S7-1500', endpoint=Signed")
logger.debug("[transport=opcua][addr=ns=2;s=Channel1.Temperature] 读: value=%.2f °C, status=Good", value)
```

### 17.7 机器视觉（UVC / CSI / GigE Vision）

```python
logger.info("[transport=uvc][addr=/dev/video0@MJPEG@1920x1080@30fps] 已打开: backend=V4L2, format=MJPG")
logger.warning("[transport=uvc][addr=/dev/video0] 帧率降级: 请求 30fps, 实际 %.1f fps (USB 带宽不足?)", actual_fps)
logger.error("[transport=uvc][addr=/dev/video0] select 超时: 摄像头无数据 (USB 断开? 驱动崩溃?)")

logger.info("[transport=gige][addr=192.168.1.50@GVSP@Basler-acA1920] 已连接: model='acA1920-155um', serial=40001234, MTU=9000")
logger.warning("[transport=gige][addr=192.168.1.50] 丢包率过高: %.2f%% (检查: 网线 / 巨型帧 / 流量控制)", loss_pct)
```

### 17.8 运动控制（步进 / 伺服 / 编码器 / PWM）

```python
# 步进电机
logger.info("[transport=pulse][addr=AXIS0@dir=DIR1,pul=PUL2] 运动开始: target=%d 脉冲 @ %.0f Hz",
            target_pulses, step_freq)
logger.warning("[transport=pulse][addr=AXIS0] 丢步检测: 编码器反馈 %d, 命令 %d, 误差 %d (≥阈值)",
               actual, commanded, abs(actual - commanded))

# 编码器
logger.debug("[transport=qenc][addr=chip=0/quad=0] 周期读取: count=%d, velocity=%.2f pulse/s", count, vel)

# PWM 舵机
logger.debug("[transport=pwm][addr=chip=0/channel=0@50Hz] 脉宽: %.0f µs (%.1f°)", pulse_us, angle)
```

### 17.9 §17 检查清单

- [ ] 每条物理接口日志都带 `[transport=xxx][addr=xxx]` 标签（标签表见 §17.1）
- [ ] 地址格式在代码库统一（Linux 设备路径 + 物理参数），README 维护地址字典
- [ ] 总线/设备枚举（启动时）记录全部结果，含负结果（缺失设备、bus-off、未授权）
- [ ] 连接/使能/打开 记录 INFO；意外断开/错误记录 WARNING 或 ERROR
- [ ] 高频物理接口（>100 Hz）禁止逐条日志；用专用子 Logger + 周期汇总 + 异常逐条
- [ ] 原子操作（pin 翻转、寄存器读写、单帧 CAN）在 DEBUG；批量操作在 INFO
- [ ] HEALTH 行（§16）扩展每条总线的关键指标（I²C NACK 数、CAN Bus-Off 数、丢帧率）
- [ ] 协议/物理层分离：底层字节流在 `transport.xxx` 子 Logger；上层业务在主 logger
- [ ] 错误消息包含可操作的恢复建议（"检查接线"、"检查上拉电阻"、"ip link set can0 up"）
- [ ] 物理参数（时钟频率、CAN 波特率、采样率）变更必须在 INFO 级别记录

---

## 18. FILE_IO 文件与存储 I/O 日志

> 全文仅 §4.4 涉及"文件 close 失败"，对配置文件/数据文件/原子写/分块/句柄泄漏/失败分类等完全不教。Agent 写文件 I/O 代码时没有规范可循。

### 18.1 统一前缀 `FILE_IO`（强制）

所有业务文件 I/O 日志必须以 `FILE_IO | op=<op> kind=<kind> path=<path>` 开头：

```python
logger.info("FILE_IO | op=open   kind=config path=/etc/myapp/config.yaml size=2.3KB")
logger.info("FILE_IO | op=read   kind=config path=/etc/myapp/config.yaml bytes=2341")
logger.info("FILE_IO | op=write  kind=data   path=/data/run_001.csv rows=100000 bytes=12.4MB dur=3.21s")
logger.info("FILE_IO | op=close  kind=data   path=/data/run_001.csv rows=100000 bytes=12.4MB")
logger.info("FILE_IO | op=rename kind=tmp     path=/tmp/x.tmp -> /data/run_001.csv")
logger.info("FILE_IO | op=delete kind=tmp     path=/tmp/x.tmp reason=committed")
```

| 字段 | 取值 |
|------|------|
| `op`   | `open` / `read` / `write` / `flush` / `fsync` / `close` / `rename` / `delete` / `lock` / `unlock` |
| `kind` | `config` / `data` / `tmp` / `state` / `lock` / `log` |
| `path` | 绝对路径更佳 |
| `bytes` | 带单位（`B`/`KB`/`MB`/`GB`） |
| `rows` | 行数（如适用） |
| `dur`  | 耗时带单位（`ms`/`s`） |

### 18.2 文件分类 `kind`

| kind | 含义 | 典型 |
|------|------|------|
| `config` | 配置文件 | YAML / JSON / TOML / INI / `.env` |
| `data` | 业务数据 | CSV / Parquet / HDF5 / NumPy / pickle / protobuf |
| `tmp` | 临时文件 | `tempfile.NamedTemporaryFile` / `*.tmp` |
| `state` | 状态/快照 | checkpoint / heartbeat / 锁文件 |
| `lock` | 锁文件 | `*.lock` / `flock` 句柄 |
| `log` | 日志自身 | 由 §3.1 / §3.2 管控 |

### 18.3 生命周期日志模式

```python
# ── 1. open ─────────────────────────────────────
try:
    f = open(path, "rb", buffering=0)
except FileNotFoundError:
    logger.error("FILE_IO | op=open kind=data path=%s FAILED reason=not_found", path); raise
except PermissionError as e:
    logger.critical("FILE_IO | op=open kind=data path=%s FAILED reason=permission_denied user=%s",
                    path, os.getuser()); raise
except OSError as e:
    logger.critical("FILE_IO | op=open kind=data path=%s FAILED errno=%d msg=%s",
                    path, e.errno, e.strerror); raise
logger.debug("FILE_IO | op=open kind=data path=%s mode=rb buffering=0", path)

# ── 2. read / write（按批次聚合）────────────────
rows = bytes_written = 0
chunk_start = time.monotonic()
BATCH = 10_000
for chunk in stream:
    f.write(chunk)
    rows += 1
    bytes_written += len(chunk)
    if rows % BATCH == 0:
        logger.debug("FILE_IO | op=write kind=data path=%s rows=%d bytes=%d dur=%.2fms",
                     path, rows, bytes_written,
                     (time.monotonic() - chunk_start) * 1000)

# ── 3. flush + fsync（强持久化场景）────────────
try:
    f.flush(); os.fsync(f.fileno())
except OSError as e:
    logger.critical("FILE_IO | op=fsync kind=data path=%s FAILED errno=%d msg=%s (durability NOT guaranteed)",
                    path, e.errno, e.strerror)

# ── 4. close ────────────────────────────────────
try:
    f.close()
except Exception:
    logger.exception("FILE_IO | op=close kind=data path=%s FAILED (handle leak suspected)", path)
```

**规则：**
- `open` 失败必须区分 `FileNotFoundError` / `PermissionError` / 其他 `OSError`
- `flush` / `fsync` 失败必须升 **CRITICAL**（数据可能丢失）
- 写循环**严禁**逐条记录；用 `rows % BATCH == 0` 取模聚合

### 18.4 原子写入模式（`tmp + os.replace`）

```python
import os, tempfile

def atomic_write(path: str, data: bytes) -> None:
    dirpath = os.path.dirname(path) or "."
    fd, tmp = tempfile.mkstemp(prefix=".tmp_", dir=dirpath)
    try:
        with os.fdopen(fd, "wb") as f:
            f.write(data)
            f.flush()
            os.fsync(f.fileno())
        os.replace(tmp, path)   # atomic on POSIX & Win ≥Vista
        logger.info("FILE_IO | op=rename kind=data path=%s -> %s bytes=%d",
                    tmp, path, len(data))
    except Exception:
        logger.exception("FILE_IO | op=write kind=data path=%s FAILED (cleaning up %s)", path, tmp)
        try: os.unlink(tmp)
        except FileNotFoundError: pass
        raise
```

**规则：**
- 任何"业务数据"落盘都必须走原子写路径
- `mkstemp` 在目标目录创建（保证 `rename` 不跨文件系统）
- `fsync` 失败不能静默吞，必须 CRITICAL
- `os.replace` 失败必须 ERROR 并清理 tmp

### 18.5 配置文件加载

启动时加载配置是 P0 关键路径：

```python
def load_config(path: str) -> dict:
    logger.info("FILE_IO | op=open kind=config path=%s", path)
    try:
        with open(path, "rb") as f:
            raw = f.read()
    except FileNotFoundError:
        logger.critical("FILE_IO | op=open kind=config path=%s FAILED reason=not_found", path); raise
    except PermissionError:
        logger.critical("FILE_IO | op=open kind=config path=%s FAILED reason=permission_denied", path); raise
    sha = hashlib.sha256(raw).hexdigest()[:12]
    logger.info("FILE_IO | op=read kind=config path=%s bytes=%d sha256=%s",
                path, len(raw), sha)
    try:
        cfg = yaml.safe_load(raw)
    except yaml.YAMLError as e:
        logger.error("FILE_IO | op=read kind=config path=%s FAILED reason=parse_error err=%s", path, e); raise
    logger.info("FILE_IO | op=close kind=config path=%s keys=%d", path, len(cfg))
    return cfg
```

**规则：**
- 配置加载失败 = **CRITICAL**（系统无法启动）
- 记录 `sha256` 前 12 位便于版本对比
- 记录 `keys=` / `sections=` 数量便于事后校验

### 18.6 大文件分块读写

```python
CHUNK = 1 << 20  # 1 MiB
total = 0
t0 = time.monotonic()
with open(src, "rb") as fin, open(dst, "wb") as fout:
    while True:
        buf = fin.read(CHUNK)
        if not buf:
            break
        fout.write(buf)
        total += len(buf)
elapsed = time.monotonic() - t0
logger.info("FILE_IO | op=copy kind=data path=%s -> %s bytes=%d dur=%.2fms throughput=%.1fMB/s",
            src, dst, total, elapsed * 1000, total / 1e6 / elapsed)
```

**规则：**
- 单次 `read()` 不得超过 64 MiB
- 循环内不记日志；循环结束后用单行汇总
- 吞吐量单位 `MB/s`（十进制）或 `MiB/s`（二进制）必须明确

### 18.7 失败模式 → 级别决策表

| 异常 / 场景 | 级别 | 前缀消息示例 |
|------------|------|--------------|
| `FileNotFoundError`（读数据） | ERROR | `op=open FAILED reason=not_found` |
| `FileNotFoundError`（写数据） | CRITICAL | 路径必现，不应找不到 |
| `PermissionError` | CRITICAL | `op=open FAILED reason=permission_denied user=...` |
| `IsADirectoryError` | ERROR | `op=open FAILED reason=is_directory` |
| `OSError(errno=28)` ENOSPC | CRITICAL | `op=write FAILED reason=disk_full free=0` |
| `OSError(errno=122)` EDQUOT | CRITICAL | 配额耗尽 |
| `UnicodeDecodeError` | ERROR | `op=read FAILED reason=encoding pos=%d` |
| `flush` / `fsync` 失败 | CRITICAL | `op=fsync FAILED durability=NOT_GUARANTEED` |
| `close` 失败 | ERROR | `op=close FAILED (handle leak suspected)` |
| `os.replace` 失败 | ERROR | `op=rename FAILED (cleanup attempted)` |
| 临时文件残留 | WARNING | `op=delete FAILED reason=residual` |
| 文件锁获取失败 | WARNING | `op=lock FAILED timeout=5s` |

### 18.8 句柄泄漏检测

在 §16 HEALTH 行中追加文件句柄指标：

```python
log.info("HEALTH | ... | fds=%d open_files=%d",
         fds_used, n_open_files)
```

**规则：**
- `open_files` 单调递增 → WARNING
- `open_files > 80%` 系统上限 → CRITICAL
- 每次进程关闭必须能在 `lsof` 输出中找到对应记录被移除

### 18.9 §18 检查清单

- [ ] 文件 I/O 日志以 `FILE_IO | op=<op> kind=<kind> path=<path>` 开头
- [ ] `kind` 取自 §18.2 的六种分类
- [ ] 所有文件路径至少出现一次（绝对路径更佳）
- [ ] 文件大小、耗时均带单位并使用 `bytes=` / `dur=` 字段
- [ ] 业务数据写入走 `tmp + os.replace` 原子写
- [ ] `fsync` 失败升 CRITICAL
- [ ] 写循环用 `rows % BATCH == 0` 取模聚合日志
- [ ] 大文件按 ≤ 64 MiB 分块
- [ ] 配置加载失败 = CRITICAL
- [ ] HEALTH 行（§16）包含 `open_files` 句柄指标
- [ ] 临时文件异常残留清理有 ERROR 记录
- [ ] 编码错误（`UnicodeDecodeError`）独立分支处理

---

**Agent 日志编程规范 全文完**

*基于 OptoSync 项目分析提取（2026-06-01）；2026-06-21 扩展 §15-§18（USER_ACTION / HEALTH & METRICS / Physical Interface / FILE_IO）*

---

## 速查表 Quick Reference

> Agent: bookmark this section. It summarizes every rule you must follow.

| # | 规则 | 违反后果 |
|---|------|----------|
| 1 | `setup_logging()` 在 `main()` 入口调用，双 handler（文件=DEBUG，控制台=INFO） | 日志丢失或生产噪音过多 |
| 2 | 批处理/守护进程使用时间戳日志文件 `{app}_{%Y%m%d_%H%M%S}.log` | 无法按运行隔离排查 |
| 3 | 使用 `logging.getLogger(__name__)`，文件格式含 `%(name)s:%(lineno)d` | 无法定位日志来源 |
| 4 | 长运行操作必须记录 START/STOP 对，附带量化指标和单位 | 不知道操作是否完成、是否正常 |
| 5 | except 块用 `logger.exception()`（自动含 traceback）；错误日志必须说明影响范围 | 堆栈丢失，无法回溯根因 |
| 6 | 使用 `%s` 延迟格式化，绝不用 f-string 写日志 | 热路径 CPU 浪费，生产环境性能下降 |
| 7 | 高频循环（>100 Hz）内不打日志，改用取模采样 | 日志文件爆炸，IO 阻塞主逻辑 |
| 8 | 绝不记录密码、令牌、密钥、PII；脱敏只留前缀或后 4 位 | 安全合规事故 |
| 9 | 使用统一前缀便于 grep：`HEALTH \|`、`AUDIT \|`、`[transport=xxx][addr=yyy]` | 日志大海捞针 |
| 10 | 长运行进程每 30-60s 输出一行 HEALTH（内存、CPU、吞吐量、传输层指标） | 无健康基线，故障发现滞后 |
| 11 | 仪器通信日志必须带 `[transport=xxx][addr=xxx]`；发现阶段记录所有扫描结果（含负结果） | 连接问题无从排查 |
| 12 | SCPI/协议流量用独立子 Logger（`logger.getChild("scpi")`），写独立文件 | 协议噪音淹没业务日志 |
| 13 | 重试时打 WARNING（含 N/M 次数），最终失败打 ERROR | 不知道重试了没有、失败原因不明 |
| 14 | 日志必须回答：谁、什么、何时、为什么、多少 —— 假设读者看不到源码 | 生产故障时束手无策 |
| 15 | 写完代码后 grep 验证：`grep "except.*pass"`、`grep "logger.*f\""`、`grep "print("` 必须无结果 | 反模式遗留到生产环境 |
| 16 | **用户触发的所有事件**必须含 `user_id`（不可逆 ID）；登录/登出/认证失败走 `audit.user` 子 Logger | 多用户系统无法按人追溯 |
| 17 | **配置变更**记录 `old= → new=`（敏感字段一律 `[REDACTED]`）；`audit` Logger 永不关闭 | 合规审计失败；安全事故无法追溯 |
| 18 | **HEALTH 行**必含 `uptime/rss/cpu/thr/q`；涉及 GPU 必须加 `gpu=idx:util\|mem\|temp\|pwr`；ML 训练静默 OOM 难以发现 |
| 19 | **写盘型守护进程**HEALTH 含 `disk:mount=pct%`；阈值按 §16.4 表（80% WARN / 90% ERR / 95% CRIT）触发 | 磁盘满导致服务静默崩溃 |
| 20 | **业务数据写入**走 `tmp + os.replace` 原子写；`fsync` 失败升 CRITICAL；`close` 失败用 `logger.exception` | 半写文件导致数据损坏；断电即丢数据 |
| 21 | **嵌入式/工业/视觉接口**（GPIO/I²C/SPI/CAN/Modbus/UVC/步进）也带 `[transport=xxx][addr=xxx]` 标签；高频（>100Hz）禁止逐条日志 | 物理层故障无从排查；日志爆 IO 阻塞 |


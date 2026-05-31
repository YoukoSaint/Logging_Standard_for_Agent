# Agent Logging Standard for Operations & Maintenance

> **Based on**: OptoSync project logging patterns  
> **Purpose**: Production-grade logging for AI agents writing maintainable, debuggable code  
> **Audience**: AI coding agents (Claude, GPT, etc.)

---

## 1. Core Principles

### 1.1 Logging Philosophy

**ALWAYS log for the operator, not the developer**

- Logs are the primary interface for diagnosing production issues
- Every log line must answer: "What happened? When? Why does it matter?"
- Assume the reader has NO access to source code during incident response

### 1.2 The Three Questions

Before writing any log statement, answer:

1. **Who needs this?** (operator, developer, auditor)
2. **When will they need it?** (real-time monitoring, post-incident analysis, compliance)
3. **What action can they take?** (if none, don't log it)

---

## 2. Log Levels (Strict Definitions)

### 2.1 Level Hierarchy

```
DEBUG < INFO < WARNING < ERROR < CRITICAL
```

### 2.2 Level Usage Rules

#### **DEBUG** — Development-time diagnostics

**When to use:**
- Function entry/exit with parameters
- Intermediate calculation results
- Protocol-level details (SCPI commands, SQL queries, API requests)

**Example from OptoSync:**
```python
logger.debug("Trying connection to %s", addr or "auto-discover")
logger.debug("Timing estimates (coarse): spec=%.0fms, keithley=%.0fms", spec_est, keithley_est)
```

**Rules:**
- NEVER log sensitive data (passwords, tokens, PII) even at DEBUG
- Use `%s` formatting, not f-strings (lazy evaluation)
- Disabled by default in production

---

#### **INFO** — Normal operational events

**When to use:**
- System state changes (started, stopped, connected, disconnected)
- Configuration changes
- Successful completion of major operations
- Performance metrics (throughput, latency, resource usage)

**Example from OptoSync:**
```python
logger.info("Connected to %s at %s", idn, addr)
logger.info("Acquisition started (target=%.1f Hz, period=%.1f ms, max=%.1f Hz)",
            target_rate, period_ms, max_rate)
logger.info("CSV logging started: %s, %s", timeseries_path, spectrum_path)
```

**Rules:**
- Include quantitative metrics (rates, counts, durations)
- Use present tense for state ("Acquisition started", not "Starting acquisition")
- Log both START and STOP of long-running operations

---

#### **WARNING** — Recoverable issues that need attention

**When to use:**
- Degraded performance (fallback to slower path)
- Retry attempts
- Resource pressure (queue filling, memory growth)
- Configuration issues that don't block operation

**Example from OptoSync:**
```python
logger.warning("Acquisition already running.")
logger.warning("Keithley busy (attempt %d/3), retrying ...", attempt + 1)
logger.warning("Acquisition thread did not stop cleanly.")
```

**Rules:**
- Always include context: what failed, why, what's being done about it
- Include retry count if applicable
- Log once per incident, not on every retry

---

#### **ERROR** — Operation failed, but system continues

**When to use:**
- Failed operations that are retried or skipped
- Data loss or corruption
- External service failures
- Unexpected exceptions (with stack trace)

**Example from OptoSync:**
```python
logger.error("Device %s measurement error: %s", dev.name, e)
logger.error("Callback error: %s", e)
logger.exception("Error closing %s CSV file", fname)  # includes stack trace
```

**Rules:**
- Use `logger.exception()` inside except blocks (auto-includes traceback)
- Include error message from exception
- Identify which component/device failed
- Log impact: "X samples dropped", "Y clients disconnected"

---

#### **CRITICAL** — System failure, immediate action required

**When to use:**
- Unrecoverable errors requiring restart
- Data integrity violations
- Security breaches
- Resource exhaustion (OOM, disk full)

**Example pattern:**
```python
logger.critical("Out of memory: RSS=%d MB exceeds limit %d MB. Shutting down.", rss, limit)
logger.critical("Database corruption detected in %s. Manual recovery required.", db_path)
```

**Rules:**
- Always followed by shutdown or failsafe mode
- Include recovery instructions in message
- Alert on-call engineer immediately

---

## 3. Structured Logging Patterns

### 3.1 Timestamped Per-Run Log Files

**Pattern from OptoSync:**
```python
ts = datetime.now().strftime("%Y%m%d_%H%M%S")
log_filepath = os.path.join(log_dir, f"{app_name}_{ts}.log")
```

**Why:**
- Each run gets isolated log file
- No log rotation complexity
- Easy to correlate logs with data files (same timestamp)
- Grep-friendly for post-incident analysis

**Agent rule:** ALWAYS use timestamped log files for batch jobs, daemons, and long-running processes.

---

### 3.2 Dual-Handler Setup (File + Console)

**Pattern from OptoSync:**
```python
# File: DEBUG level, full context
file_handler.setLevel(logging.DEBUG)
file_handler.setFormatter(logging.Formatter(
    "%(asctime)s [%(levelname)-7s] %(name)s:%(lineno)d - %(message)s"
))

# Console: INFO level, concise
console_handler.setLevel(logging.INFO)
console_handler.setFormatter(logging.Formatter(
    "%(asctime)s [%(levelname)-7s] %(message)s"
))
```

**Why:**
- Operators see clean INFO+ messages in terminal
- Full DEBUG trace available in file for debugging
- File includes module name + line number for code navigation

**Agent rule:** Always configure both handlers with different levels and formats.

---

### 3.3 Specialized Loggers for Protocols

**Pattern from OptoSync:**
```python
# Separate logger for SCPI traffic (only when SYNC_DEBUG=1)
scpi_logger = logging.getLogger("scpi")
scpi_handler = RotatingFileHandler(f"{app_name}_{ts}_scpi.log")
scpi_logger.addHandler(scpi_handler)
```

**Why:**
- Protocol dumps are verbose and specialized
- Separate file prevents pollution of main log
- Can be enabled/disabled independently

**Agent rule:** Use named child loggers for:
- Protocol traffic (SQL, HTTP, SCPI, MQTT)
- Performance profiling
- Audit trails

---

## 4. Operational Logging Patterns

### 4.1 State Transitions

**Pattern:**
```python
# START
logger.info("Acquisition started (target=%.1f Hz, period=%.1f ms, max=%.1f Hz)",
            target_rate, period_ms, max_rate)

# STOP
logger.info("Acquisition stopped: %d points in %.1f s (target=%.1f Hz, actual=%.1f Hz, dropped=%d)",
            count, elapsed, target_rate, actual_rate, dropped)
```

**Rules:**
- Log START with configuration
- Log STOP with summary statistics
- Include performance metrics (throughput, error rate)

---

### 4.2 Health Monitoring

**Pattern from OptoSync:**
```python
log.info("HEALTH | uptime=%ds | RSS=%.1fMB (Δ%.1f) | CPU=%.1f%% | thr=%d | acq=%dsmp @%.1fHz | stream=%s q=%d/%d",
         uptime, rss_mb, rss_delta, cpu_pct, thread_count, sample_count, rate, stream_status, queue_depth, queue_max)
```

**Rules:**
- Use consistent prefix ("HEALTH |") for grep-ability
- Include all key metrics in one line
- Log periodically (every 30-60s for long-running processes)
- Include deltas (Δ) for trending

---

### 4.3 Adaptive Behavior

**Pattern from OptoSync:**
```python
logger.info("Adaptive rate control: actual tick %.0f ms > target %.0f ms. Settling at %.1f Hz.",
            actual_ms, target_ms, adapted_hz)
```

**Rules:**
- Log when system auto-adjusts parameters
- Explain WHY (actual > target)
- Show WHAT changed (new rate)
- Use "Adaptive", "Fallback", "Throttling" keywords for grep

---

### 4.4 Resource Cleanup

**Pattern from OptoSync:**
```python
try:
    file.close()
except Exception:
    logger.exception("Error closing %s CSV file", fname)
```

**Rules:**
- Always log cleanup failures (they indicate resource leaks)
- Use `logger.exception()` for stack trace
- Include resource identifier (file name, connection ID)

---

## 5. Performance-Aware Logging

### 5.1 Hot Path Logging

**Anti-pattern:**
```python
# BAD: logs on every sample (10,000/sec)
for sample in stream:
    logger.debug("Processing sample %d", sample.id)
```

**Correct pattern:**
```python
# GOOD: log every Nth sample or on error only
if sample_count % 1000 == 0:
    logger.debug("Processed %d samples", sample_count)
```

**Rules:**
- NEVER log inside tight loops (>100 Hz)
- Use modulo sampling for high-frequency events
- Move DEBUG logs outside hot path

---

### 5.2 Lazy Formatting

**Anti-pattern:**
```python
# BAD: f-string evaluated even if DEBUG disabled
logger.debug(f"Data: {expensive_repr(obj)}")
```

**Correct pattern:**
```python
# GOOD: % formatting deferred until needed
logger.debug("Data: %s", expensive_repr(obj))
```

**Rules:**
- Use `%s` formatting, not f-strings
- Logger evaluates arguments only if level enabled
- Saves CPU in production (DEBUG disabled)

---

## 6. Error Handling & Logging

### 6.1 Exception Logging

**Pattern from OptoSync:**
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

**Rules:**
- Catch specific exceptions first
- Use `logger.exception()` for unexpected errors (includes traceback)
- Log impact (dropped samples, failed requests)
- Re-raise if unrecoverable

---

### 6.2 Retry Logging

**Pattern from OptoSync:**
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

**Rules:**
- Log WARNING on retry
- Log ERROR on final failure
- Include attempt count (N/M format)
- Don't log every retry (use exponential backoff)

---

## 7. Context & Correlation

### 7.1 Request/Session IDs

**Pattern:**
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

**Rules:**
- Prefix all logs with `[request_id]` or `[session_id]`
- Enables grep-based request tracing
- Essential for multi-threaded/async systems

---

### 7.2 Component Identification

**Pattern from OptoSync:**
```python
logger = logging.getLogger(__name__)  # e.g., "drivers.keithley_driver"

logger.info("Connected to %s at %s", idn, addr)
# Output: 2026-05-31 17:45:23 [INFO   ] drivers.keithley_driver:95 - Connected to ...
```

**Rules:**
- Use `logging.getLogger(__name__)` for automatic module naming
- Include line number in file handler format
- Enables quick code navigation during debugging

---

## 8. Metrics & Observability

### 8.1 Quantitative Logging

**Pattern from OptoSync:**
```python
logger.info("Acquisition stopped: %d points in %.1f s (target=%.1f Hz, actual=%.1f Hz, dropped=%d)",
            count, elapsed, target_rate, actual_rate, dropped)
```

**Rules:**
- Always include units (Hz, MB, ms, %)
- Log both target and actual for comparison
- Include error/drop counts
- Use consistent precision (%.1f for rates, %.3f for latency)

---

### 8.2 Periodic Metrics

**Pattern from OptoSync:**
```python
# Every 30 seconds
log.info("HEALTH | uptime=%ds | RSS=%.1fMB (Δ%.1f) | CPU=%.1f%% | ...")
```

**Rules:**
- Use fixed interval (30-60s)
- Include deltas (Δ) for trending
- Use consistent prefix for parsing
- Log to separate metrics file if high volume

---

## 9. Security & Compliance

### 9.1 Never Log Sensitive Data

**Forbidden:**
```python
# NEVER DO THIS
logger.debug("API key: %s", api_key)
logger.info("User password: %s", password)
logger.debug("Credit card: %s", cc_number)
```

**Correct:**
```python
logger.debug("API key: %s", api_key[:8] + "..." if api_key else None)
logger.info("User authenticated: %s", username)
logger.debug("Payment processed: last4=%s", cc_last4)
```

**Rules:**
- Redact passwords, tokens, keys, PII
- Log only non-sensitive identifiers (username, user_id, last4)
- Use `[REDACTED]` placeholder for audit trails

---

### 9.2 Audit Logging

**Pattern:**
```python
audit_logger = logging.getLogger("audit")
audit_logger.info("USER_ACTION | user=%s | action=%s | resource=%s | result=%s",
                  user_id, action, resource_id, "SUCCESS" if ok else "DENIED")
```

**Rules:**
- Use separate logger for audit trail
- Include: who, what, when, where, result
- Never disable audit logs (even in production)
- Immutable append-only log file

---

## 10. Agent Implementation Checklist

When writing code, agents MUST:

- [ ] Configure logging in `main()` or module entry point
- [ ] Use timestamped log files for batch/daemon processes
- [ ] Set up dual handlers (file=DEBUG, console=INFO)
- [ ] Use `logging.getLogger(__name__)` in every module
- [ ] Log START/STOP of long-running operations with metrics
- [ ] Use `logger.exception()` in except blocks
- [ ] Include units in all quantitative logs
- [ ] Avoid logging in hot paths (>100 Hz loops)
- [ ] Use `%s` formatting, not f-strings
- [ ] Never log passwords, tokens, or PII
- [ ] Add health monitoring for daemons (RSS, CPU, throughput)
- [ ] Use consistent prefixes for grep-ability (HEALTH, ALERT, AUDIT)

---

## 11. Example: Complete Logging Setup

```python
"""
Example: Production-grade logging setup for an agent-written application.
"""
import logging
import logging.handlers
import os
from datetime import datetime

def setup_logging(app_name="MyApp", log_dir="logs", console_level=logging.INFO):
    """Configure production logging with timestamped files."""
    os.makedirs(log_dir, exist_ok=True)
    
    # Timestamped log file
    ts = datetime.now().strftime("%Y%m%d_%H%M%S")
    log_path = os.path.join(log_dir, f"{app_name}_{ts}.log")
    
    # Root logger
    root = logging.getLogger()
    root.setLevel(logging.DEBUG)
    root.handlers.clear()
    
    # File handler: DEBUG level, full context
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
    
    # Console handler: INFO level, concise
    console_handler = logging.StreamHandler()
    console_handler.setLevel(console_level)
    console_handler.setFormatter(logging.Formatter(
        "%(asctime)s [%(levelname)-7s] %(message)s"
    ))
    root.addHandler(console_handler)
    
    logging.info("Logging initialized: %s", log_path)
    return log_path


# Usage in modules
logger = logging.getLogger(__name__)

def process_data(data_id, payload):
    logger.info("[%s] Processing started: %d bytes", data_id, len(payload))
    try:
        result = expensive_operation(payload)
        logger.info("[%s] Processing completed: %d results in %.2f s",
                    data_id, len(result), elapsed)
        return result
    except ValueError as e:
        logger.error("[%s] Validation failed: %s", data_id, e)
        raise
    except Exception:
        logger.exception("[%s] Unexpected error", data_id)
        raise
```

---

## 12. Anti-Patterns to Avoid

### ❌ Print statements instead of logging
```python
print("Starting process...")  # BAD
logger.info("Starting process...")  # GOOD
```

### ❌ Logging without context
```python
logger.error("Failed")  # BAD: what failed?
logger.error("Device %s connection failed: %s", device_id, error)  # GOOD
```

### ❌ Logging in tight loops
```python
for i in range(10000):
    logger.debug("Processing %d", i)  # BAD: 10k log lines
```

### ❌ F-strings in log calls
```python
logger.debug(f"Value: {expensive_call()}")  # BAD: always evaluated
logger.debug("Value: %s", expensive_call())  # GOOD: lazy evaluation
```

### ❌ Swallowing exceptions silently
```python
try:
    risky_operation()
except Exception:
    pass  # BAD: silent failure
```

### ❌ Logging sensitive data
```python
logger.info("Password: %s", password)  # BAD: security violation
logger.info("User authenticated: %s", username)  # GOOD
```

---

## 13. Summary: The Golden Rules

1. **Log for operators, not developers** — assume no source code access
2. **Use structured formats** — timestamped files, consistent prefixes
3. **Include context** — who, what, when, why, how much
4. **Respect performance** — avoid hot paths, use lazy formatting
5. **Never log secrets** — redact passwords, tokens, PII
6. **Log state changes** — START/STOP with metrics
7. **Use appropriate levels** — DEBUG/INFO/WARNING/ERROR/CRITICAL
8. **Enable traceability** — request IDs, module names, line numbers
9. **Monitor health** — periodic metrics for long-running processes
10. **Test your logs** — grep for patterns, verify readability

---

**End of Agent Logging Standard**

*Generated from OptoSync project analysis (2026-05-31)*

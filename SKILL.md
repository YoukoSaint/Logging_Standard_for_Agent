---
name: agent-logging-standard
description: Production-grade logging standards for AI agents writing maintainable, ops-friendly code. Covers log levels, structured patterns, health monitoring, error handling, and security compliance. Use when writing or reviewing any code that produces logs.
origin: Log_Standard_for_Agent
---

> ⚠️ This Skill is a CONDENSED reference. For the COMPLETE specification with full rationale, examples, and anti-patterns, read AGENT_LOGGING_STANDARD_zh.md.

# Agent Logging Standard

Production-grade logging patterns extracted from the OptoSync hardware acquisition system,
refined for general use. Write logs that help operators diagnose issues **without access to
source code**.

## When to Activate

- Writing or reviewing any code that emits logs
- Setting up logging configuration for a new project
- Refactoring existing log statements
- User asks for "production-grade logging" or "ops-friendly logs"
- Building daemons, services, CLI tools, or batch jobs

## Scope Boundaries

Activate for:
- Python `logging` module configuration and usage
- Log level selection (DEBUG / INFO / WARNING / ERROR / CRITICAL)
- Structured log formats, health monitoring, error handling patterns
- Security boundaries (never log secrets)
- Instrument transport layer tagging and lifecycle tracking
- Protocol/transport layer separation (SCPI vs TCP/UART)

Do NOT use as primary source for:
- Tracing/metrics SDKs (OpenTelemetry, Prometheus) — see `dashboard-builder`
- Structured JSON logging pipelines (ELK, Loki)
- Framework-specific log adapters (Django, FastAPI)

---

## 1. Level Decision Table (ALWAYS consult first)

| Situation | Level | Example Message |
|-----------|-------|-----------------|
| Variable value, function entry/exit, protocol detail | `DEBUG` | `"Query returned %d rows in %.2f ms"` |
| Service start/stop, config change, operation done | `INFO` | `"Server listening on :8080 (workers=4)"` |
| Auto-degrade, retry, resource near limit | `WARNING` | `"Redis unreachable, using local cache (attempt 2/3)"` |
| Request failed, data corrupt, external service down | `ERROR` | `"Payment gateway timeout: order_id=%s"` |
| OOM, disk full, security breach, irreversible loss | `CRITICAL` | `"Disk 98% full on /data, shutdown imminent"` |

## 2. Logging Setup (MANDATORY pattern)

Every project entry point MUST configure logging like this:

```python
import logging
import logging.handlers
import os
from datetime import datetime

def setup_logging(app_name="MyApp", log_dir="logs", console_level=logging.INFO):
    """Configure production logging with timestamped per-run files."""
    os.makedirs(log_dir, exist_ok=True)

    ts = datetime.now().strftime("%Y%m%d_%H%M%S")
    log_path = os.path.join(log_dir, f"{app_name}_{ts}.log")

    root = logging.getLogger()
    root.setLevel(logging.DEBUG)
    root.handlers.clear()

    # File: DEBUG level, module + line number for code navigation
    fh = logging.handlers.RotatingFileHandler(
        log_path, maxBytes=10 * 1024 * 1024, backupCount=7, encoding="utf-8",
    )
    fh.setLevel(logging.DEBUG)
    fh.setFormatter(logging.Formatter(
        "%(asctime)s [%(levelname)-7s] %(name)s:%(lineno)d - %(message)s"
    ))
    root.addHandler(fh)

    # Console: INFO level, concise
    ch = logging.StreamHandler()
    ch.setLevel(console_level)
    ch.setFormatter(logging.Formatter(
        "%(asctime)s [%(levelname)-7s] %(message)s"
    ))
    root.addHandler(ch)

    logging.info("Logging initialized: %s", log_path)
    return log_path
```

Key decisions embedded in this setup:
- **Timestamped files** — every run gets its own file, no rotation complexity
- **Dual handlers** — operators see clean INFO on console, full DEBUG in file
- **Line numbers in file log** — `%(name)s:%(lineno)d` enables instant code navigation
- **Clears existing handlers** — safe to call multiple times (GUI reload scenarios)

## 3. Module-Level Logger

Every `.py` file that logs MUST use:

```python
import logging
logger = logging.getLogger(__name__)
```

Never: `logger = logging.getLogger("hardcoded_name")` — loses module traceability.

## 4. State Transition Logging

Every long-running operation MUST log START and STOP with metrics:

```python
# START — include configuration
logger.info("Acquisition started (target=%.1f Hz, period=%.1f ms)", rate, period_ms)

# STOP — include summary statistics (target vs actual, errors)
logger.info("Acquisition stopped: %d points in %.1f s (actual=%.1f Hz, dropped=%d)",
            count, elapsed, actual_rate, dropped)
```

## 5. Error Handling Rules

### Expected exceptions → ERROR, no stack trace
```python
try:
    device.measure()
except DeviceError as e:
    logger.error("Device %s measurement error: %s", dev.name, e)
    dropped += 1
```

### Unexpected exceptions → logger.exception() (includes traceback)
```python
except Exception:
    logger.exception("Unexpected error in measurement loop")
    raise
```

### Retry → WARNING on retry, ERROR on final failure
```python
for attempt in range(3):
    try:
        return operation()
    except TransientError as e:
        if attempt < 2:
            logger.warning("Attempt %d/3 failed: %s", attempt + 1, e)
            time.sleep(backoff)
        else:
            logger.error("Operation failed after 3 attempts: %s", e)
            raise
```

## 6. Health Monitoring (daemons / long-running processes)

For any process running longer than a few minutes, add periodic health logging:

```python
# Every 30-60 seconds, ONE line with all key metrics
log.info("HEALTH | uptime=%ds | RSS=%.1fMB (Δ%.1f) | CPU=%.1f%% | thr=%d | "
         "acq=%dsmp @%.1fHz | q=%d/%d",
         uptime, rss_mb, rss_delta, cpu_pct, thread_count, rate, queue_depth, queue_max)
```

Rules:
- **Consistent prefix** (e.g., `HEALTH |`) for grep
- **Include deltas** (Δ) for trend detection — RSS growing by 50 MB/h = leak
- **Alert thresholds**: log WARNING when RSS growth > 50 MB/h, ERROR when RSS > 500 MB

## 7. Performance Rules

### NEVER log inside hot paths (>100 Hz loops)
```python
# WRONG — logs 10,000 times per second
for sample in stream:
    logger.debug("Processing sample %d", sample.id)

# RIGHT — modulo sampling
for sample in stream:
    if sample_count % 1000 == 0:
        logger.debug("Processed %d samples", sample_count)
```

### NEVER use f-strings in log calls
```python
# WRONG — evaluated even when DEBUG disabled
logger.debug(f"Data: {expensive_repr(obj)}")

# RIGHT — lazy evaluation
logger.debug("Data: %s", expensive_repr(obj))
```

## 8. Security: NEVER Log These

Forbidden in ALL log levels including DEBUG:
- Passwords, API keys, tokens
- Credit card numbers, SSN, PII
- Session cookies, auth headers

```python
# WRONG
logger.debug("API key: %s", api_key)

# RIGHT
logger.debug("API key: %s...", api_key[:8])
logger.info("User authenticated: %s", username)  # non-sensitive identifier only
```

## 9. Audit Trail (optional, useful for multi-user systems)

```python
audit = logging.getLogger("audit")
audit.info("USER_ACTION | user=%s | action=%s | resource=%s | result=%s",
           user_id, action, resource_id, "SUCCESS" if ok else "DENIED")
```

Separate logger → separate file → immutable history.

## 10. Agent Checklist (verify before marking code complete)

- [ ] `setup_logging()` called in entry point with timestamped file
- [ ] File handler = DEBUG, console handler = INFO
- [ ] Every module uses `logging.getLogger(__name__)`
- [ ] START/STOP logged for long-running operations with metrics
- [ ] `logger.exception()` used in except blocks for unexpected errors
- [ ] All quantitative logs include units (Hz, MB, ms, %)
- [ ] No log calls inside >100 Hz loops
- [ ] `%s` formatting used (not f-strings)
- [ ] No passwords, tokens, or PII in any log
- [ ] Health monitoring added for daemon processes
- [ ] Consistent prefixes for grep (HEALTH |, ALERT |, AUDIT |)



## 11. Instrument Automation Transport Layer

For multi-device automation where instruments communicate over USB, Ethernet/IP,
Serial, or GPIB. Every log line must identify **how** a device was talked to.

### Transport Tagging
```python
# ALWAYS tag instrument communication with [transport=xxx][addr=xxx]
logger.info("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] Connected")
logger.error("[transport=tcp][addr=192.168.1.100:5025] Command timeout")
logger.warning("[transport=serial][addr=COM3@115200] Buffer overflow")
```

### Connection Lifecycle
```python
# INFO: connect/disconnect/reconnect
logger.info("[transport=tcp][addr=192.168.1.100:5025] "
            "Reconnected: was down for 12.3s (3 attempts)")

# WARNING: unexpected disconnection
logger.warning("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
               "Connection lost: device removed from bus")
```

### Health Metrics
```python
# Add per-transport metrics to every HEALTH line
log.info("HEALTH | tcp:lat=%.1fms|retx=%d | usb:tx=%dKB/s|err=%d",
         tcp_lat, tcp_retx, usb_kbps, usb_err)
```

### Protocol/Transport Separation
```python
logger = logging.getLogger("controller")
scpi_log = logging.getLogger("controller.scpi")      # SCPI commands
transport_log = logging.getLogger("controller.transport")  # raw bytes
```

### Discovery Logging
```python
# Log every transport scanned, including negatives
logger.info("[discover][tcp] Probing 192.168.1.0/24:5025...")
logger.warning("[discover][tcp] No devices found (timeout=5s)")
```

## Agent Checklist Additions (instrument automation)

- [ ] Every instrument log line tagged with `[transport=xxx][addr=xxx]`
- [ ] Connection/disconnection logged at INFO with address
- [ ] HEALTH line includes per-transport metrics
- [ ] SCPI and transport layers use separate child loggers
- [ ] Discovery logs every scanned transport (including empty results)
- [ ] Error messages include actionable recovery hints per transport type

## Anti-Patterns Quick Reference

| ❌ Wrong | ✅ Right |
|----------|----------|
| `print("starting...")` | `logger.info("starting...")` |
| `logger.error("Failed")` | `logger.error("Device %s: %s", name, err)` |
| `logger.debug(f"{x}")` | `logger.debug("%s", x)` |
| `except: pass` | `except Exception: logger.exception(...)` |
| `logger.info("pwd=%s", pwd)` | `logger.info("user=%s auth=OK", user)` |

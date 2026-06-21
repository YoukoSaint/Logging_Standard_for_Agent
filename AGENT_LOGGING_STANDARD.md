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

> Full specification in **§16 HEALTH & METRICS** (GPU/VRAM, disk, FDs, network, coroutines, cgroup, threshold table, collection template).

**Pattern from OptoSync (minimal):**
```python
log.info("HEALTH | uptime=%ds | RSS=%.1fMB (Δ%.1f) | CPU=%.1f%% | thr=%d | acq=%dsmp @%.1fHz | stream=%s q=%d/%d",
         uptime, rss_mb, rss_delta, cpu_pct, thread_count, sample_count, rate, stream_status, queue_depth, queue_max)
```

**Rules:**
- Use consistent prefix ("HEALTH |") for grep-ability
- Include all key metrics in one line
- Log periodically (every 30-60s for long-running processes; short tasks skip HEALTH, see §16.7)
- Include deltas (Δ) for trending
- For GPU/ML workloads, MUST extend with `gpu=idx:util|mem|temp|pwr` (§16.2)

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

> For multi-user systems, must include `user_id` context. Full specification in **§15 USER_ACTION**.

**Pattern:**
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

**Rules:**
- Prefix all user-triggered logs with `[user=<uid>][session=<sid>][req=<rid>]` triple
- System-internal events (cron / background) use `[user=system][session=-][req=<rid>]`
- `user_id` MUST be an **internal non-reversible ID** (never email, phone, real name)
- Enables grep-based tracing: (1) all actions by a user, (2) full call chain of a request, (3) all interactions in a session
- In multi-threaded/async systems the three IDs MUST be propagated via `contextvars`
- Never silently omit any of the three; use placeholders like `[user=unknown]`

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

> Full specification in **§15 USER_ACTION** (field dictionary, login lifecycle, config changes, cancellations, validation failures, permission denials, data-sensitive operations).

**Pattern (minimal):**
```python
audit_logger = logging.getLogger("audit")
audit_logger.info("USER_ACTION | user=%s | action=%s | resource=%s | result=%s",
                  user_id, action, resource_id, "SUCCESS" if ok else "DENIED")
```

**Rules:**
- Use separate logger for audit trail (recommended child loggers: `audit.user`, `audit.auth`, `audit.config`, `audit.permission`)
- Include: who, what, when, where, result
- Never disable audit logs (even in production)
- Immutable append-only log file
- `user` field uses an **internal non-reversible ID** (not email, phone, real name)
- Sensitive values (old/new config, user input) MUST be `[REDACTED]`

---

## 10. Agent Implementation Checklist

When writing code, agents MUST:

### Core setup (§3)

- [ ] Configure logging in `main()` or module entry point
- [ ] Use timestamped log files for batch/daemon processes
- [ ] Set up dual handlers (file=DEBUG, console=INFO)
- [ ] Use `logging.getLogger(__name__)` in every module
- [ ] Use consistent prefixes for grep-ability (`HEALTH |`, `ALERT |`, `USER_ACTION |`, `FILE_IO |`, `[transport=xxx][addr=xxx]`)

### Logging content (§4 / §6 / §8)

- [ ] Log START/STOP of long-running operations with metrics
- [ ] Use `logger.exception()` in except blocks
- [ ] Include units in all quantitative logs (Hz, MB, ms, %, °C, W)

### Performance & security (§5 / §9)

- [ ] Avoid logging in hot paths (>100 Hz loops)
- [ ] Use `%s` formatting, not f-strings
- [ ] Never log passwords, tokens, or PII

### User interaction (§15)

- [ ] All user-triggered events include `user_id` (non-reversible ID)
- [ ] Login/logout/auth failures logged in separate `audit.user` logger
- [ ] Config changes record `old=` → `new=` (sensitive fields `[REDACTED]`)
- [ ] User cancellation (INFO) vs system crash (CRITICAL) strictly distinguished
- [ ] Input validation failures record field name + reason, value `[REDACTED]`
- [ ] Permission denials include `user_id` + `required_role` + `resource`
- [ ] Data export/delete/download records `rows=` / `format=` / `confirmation=2FA`

### System resources & periodic metrics (§16)

- [ ] Add health monitoring (HEALTH line) for daemons
- [ ] HEALTH line includes the five required fields: `uptime / rss / cpu / thr / q`
- [ ] GPU workloads record `gpu=idx:util|mem|temp|pwr`
- [ ] High-concurrency services add `fds` + `net:est|tw|cw`
- [ ] Disk-writing daemons add `disk:mount=pct%`
- [ ] First HEALTH output ≥ 5s after start; `final=true` line on shutdown
- [ ] Threshold alerts per §16.4 table (WARNING/ERROR/CRITICAL)

### Physical interfaces (§14 / §17)

- [ ] Every instrument log line tagged with `[transport=xxx][addr=xxx]`
- [ ] Connect/disconnect logged at INFO with address and device identity
- [ ] Unexpected disconnects at WARNING or ERROR
- [ ] SCPI protocol and transport use separate child loggers
- [ ] Embedded/industrial/vision interfaces (§17) use corresponding transport tags
- [ ] High-frequency physical interfaces (>100 Hz) use child logger + periodic summary, never per-event
- [ ] Startup records bus/device enumeration results (including negatives)

### File I/O (§18)

- [ ] File I/O logs begin with `FILE_IO | op=<op> kind=<kind> path=<path>`
- [ ] Business data writes go through `tmp + os.replace` atomic write
- [ ] `fsync` failure escalates to CRITICAL
- [ ] Write loops aggregate logs with `rows % BATCH == 0`
- [ ] Large files read in chunks ≤ 64 MiB
- [ ] Config load failure = CRITICAL
- [ ] Leftover temp files generate ERROR on cleanup failure

> Full per-dimension checklists: §15.8 / §16.9 / §17.9 / §18.9.

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

### ❌ User actions without `user_id` (violates §15)
```python
logger.error("Permission denied")                                            # BAD: no user/resource context
logger.error("AUTHZ_DENIED | user=%s | resource=%s | required_role=%s",      # GOOD
             user_id, resource_id, required_role)
```

### ❌ Per-event logging on high-frequency I/O (violates §5.1 / §17.4)
```python
for msg in can_bus:                                                         # 1kHz → log explosion
    logger.info("[transport=can] recv 0x%03X", msg.arbitration_id)
# GOOD: see §17.4 — child logger + periodic summary + per-event on errors
```

### ❌ `f.read()` of GB-scale files (violates §18.6)
```python
data = open("big.bin", "rb").read()    # OOM
# GOOD: chunked reads ≤ 64 MiB
```

### ❌ `open(..., 'w')` overwriting production data (violates §18.4)
```python
open("/data/run.csv", "w").write(...)   # partial writes corrupt data
# GOOD: tmp + os.replace atomic write
```

### ❌ `try/except: pass` swallowing `fsync` failure (violates §18.3)
```python
try: os.fsync(fd)
except: pass                           # power loss = data loss
# GOOD: CRITICAL level with durability=NOT_GUARANTEED
```

### ❌ No GPU/VRAM monitoring in ML workloads (violates §16)
```python
log.info("HEALTH | rss=%.1fMB cpu=%.1f%%", rss, cpu)    # silent OOM during training
# GOOD: include gpu=idx:util|mem|temp|pwr (see §16.2)

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
11. **Tag by dimension** — every line must be bucketable: `USER_ACTION |`, `HEALTH |`, `FILE_IO |`, `[transport=xxx][addr=xxx]` (see §15-§18)

---



---

## 14. Instrument Automation Transport Layer

> For multi-device automation systems where instruments communicate over heterogeneous
> transports (USB, Ethernet/IP, Serial, GPIB). Every log line must identify **how** a
> device was talked to, because the failure mode and debugging path differ by transport.

### 14.1 Transport Tagging (MANDATORY)

Every log line involving instrument communication MUST include a transport tag:

```python
logger.info("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] Connected")
logger.error("[transport=tcp][addr=192.168.1.100:5025] Command timeout after 5s")
logger.warning("[transport=serial][addr=COM3@115200] Buffer overflow, purging")
```

**Tag format:**

| Transport | Tag | Address Format |
|-----------|-----|----------------|
| USB (USBTMC/VISA) | `transport=usb` | `USB0::VID::PID::serial::INSTR` |
| TCP/IP (VXI-11, Raw Socket) | `transport=tcp` | `host:port` |
| Serial / RS-232 | `transport=serial` | `port@baud` |
| GPIB | `transport=gpib` | `GPIB0::pad` |
| Unknown / auto-discovered | `transport=auto` | Include discovery method |

**Rules:**
- Always use `[transport=xxx][addr=yyy]` as the first two fields after the log level
- Keep the address format consistent so `grep addr=` works across the codebase
- When an instrument is reachable via multiple paths (e.g., USB and TCP), tag the active one

---

### 14.2 Connection Lifecycle

Every connection, disconnection, and reconnection MUST be logged at **INFO** level:

```python
# Connect
logger.info("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
            "Connected: idn='Keithley 2450, 4430138, v3.0'")

# Disconnect (expected)
logger.info("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
            "Disconnected: user-initiated close")

# Disconnect (unexpected - USB hot-plug / cable pull)
logger.warning("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
               "Connection lost: device removed from bus")

# Auto-reconnect
logger.info("[transport=tcp][addr=192.168.1.100:5025] "
            "Reconnected: was down for 12.3s (3 attempts)")

# Reconnect failure
logger.error("[transport=tcp][addr=192.168.1.100:5025] "
             "Reconnect failed after 5 attempts (last error: Connection refused)")
```

**Connection health events at WARNING level:**

```python
# USB bus reset
logger.warning("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
               "USB bus reset detected, re-acquiring handle...")

# TCP keepalive failure
logger.warning("[transport=tcp][addr=192.168.1.100:5025] "
               "TCP keepalive timed out, probing connection...")

# Serial framing error
logger.warning("[transport=serial][addr=COM3@115200] "
               "Serial framing error on byte 0x%02x, resyncing...")
```

---

### 14.3 Transport Metrics in Health Monitoring

Extend the periodic HEALTH line (Section 4.2) with per-transport metrics:

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

**Key transport metrics per type:**

| Transport | Metrics |
|-----------|---------|
| TCP | latency (ms), retransmission count, connection pool usage, socket timeout count |
| USB | throughput (KB/s), transfer error count, bus reset count |
| Serial | buffer fill level, framing error count, break signal count |
| GPIB | timeout count, EOS error count, bus contention count |

**Rules:**
- Zero errors is normal - log at DEBUG; first error in a period -> WARNING
- Track deltas since last HEALTH line (not cumulative)
- Include transport metrics in every HEALTH line, even when zero

---

### 14.4 Protocol / Transport Layer Separation

Log SCPI (or any instrument protocol) traffic on a **child logger** separate from
transport-layer logs:

```python
# Logger hierarchy:
#   "controller"           -> business logic (main log)
#   "controller.scpi"      -> SCPI commands sent to instruments
#   "controller.transport" -> TCP/UART raw bytes, USB bulk transfers

logger = logging.getLogger("controller")
scpi_log = logging.getLogger("controller.scpi")
transport_log = logging.getLogger("controller.transport")
```

**Example usage:**

```python
# Business-level log
logger.info("[transport=tcp][addr=192.168.1.100:5025] Measure voltage on channel 1")

# SCPI log (separate file, enabled via SCPI_DEBUG=1)
scpi_log.debug("-> *IDN?")
scpi_log.debug("<- Keithley 2450, 4430138, v3.0")

# Transport log (separate file, enabled via TRANS_DEBUG=1)
transport_log.debug("SENT: 7 bytes via TCP stream")
transport_log.debug("RECV: 41 bytes in 2.3ms (1 TCP segment)")
```

**Benefits:**
- Main log stays clean - protocol noise does not mask application events
- SCPI log is grep-able for "what command caused the instrument to error"
- Transport log reveals low-level issues (partial reads, TCP fragmentation)

---

### 14.5 Instrument Discovery Logging

Auto-discovery is the most fragile and hardest-to-debug phase of instrument automation.
Log it thoroughly:

```python
# Discovery start
logger.info("[discover][usb] Scanning USB for USBTMC instruments...")
logger.info("[discover][tcp] Probing subnet 192.168.1.0/24 on port 5025...")

# Discovery results
logger.info("[discover][usb] Found 3 instruments: "
            "USB0::0x05E6::0x2450::4430138::INSTR (Keithley 2450), "
            "USB0::0x1AB1::0x0588::DG4162-12345::INSTR (Rigol DG4162)")

logger.info("[discover][tcp] Found 2 instruments: "
            "192.168.1.100:5025 (KEITHLEY 2450), "
            "192.168.1.101:5025 (RIGOL DG4162)")

# Discovery failure (most important to log)
logger.warning("[discover][tcp] mDNS query for 'KEITHLEY*' returned 0 devices (timeout=5s)")
logger.warning("[discover][usb] No USBTMC devices found on USB bus (check: driver? power?)")

# Name resolution and matching
logger.warning("[discover] Config requested 'spectrometer' but no exact match found; "
               "closest USB device: USB0::0x1AB1::0x0588::DG4162-12345::INSTR")
```

**Rules:**
- Log every transport scanned, even if nothing found (negative results matter)
- Include full instrument identification string for post-mortem matching
- Log mismatches and fallback logic - these are the #1 source of "works on my bench" bugs

---

### 14.6 Transport Error Classification

Different transports produce different errors. Log them in a way that
enables transport-specific alerting:

```python
# TCP errors
logger.error("[transport=tcp][addr=192.168.1.100:5025] "
             "Connection refused (target port closed or instrument off)")
logger.error("[transport=tcp][addr=10.0.0.50:5025] "
             "Connection timed out (no route to host? wrong subnet?)")
logger.warning("[transport=tcp][addr=192.168.1.100:5025] "
               "Partial read: expected 1024 bytes, got 312 (TCP fragmentation)")

# USB errors
logger.error("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
             "USB transfer failed: LIBUSB_ERROR_PIPE (stalled endpoint)")
logger.error("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
             "USB transfer failed: LIBUSB_ERROR_NO_DEVICE (cable disconnected)")
logger.warning("[transport=usb][addr=USB0::0x05E6::0x2450::4430138::INSTR] "
               "USB bandwidth exceeded: requested 512 bytes, max packet=64")

# Serial errors
logger.error("[transport=serial][addr=COM3@115200] "
             "Serial timeout: no response for 5s (wrong baud rate?)")
logger.warning("[transport=serial][addr=COM3@115200] "
               "Serial overrun error: 32 bytes lost (flow control issue?)")

# GPIB errors
logger.error("[transport=gpib][addr=GPIB0::12] "
             "GPIB timeout: no SRQ within 10s")
logger.warning("[transport=gpib][addr=GPIB0::12] "
               "GPIB EOS error: unexpected terminator byte 0x%02x")
```

---

### 14.7 Transport-Layer Agent Checklist

When writing instrument automation code, agents MUST additionally check:

- [ ] Every instrument log line tagged with `[transport=xxx][addr=xxx]`
- [ ] Connection/disconnection logged at INFO with address and device identity
- [ ] Unexpected disconnections logged at WARNING or ERROR
- [ ] HEALTH line includes per-transport metrics (latency, errors, throughput)
- [ ] SCPI protocol and transport layer use separate child loggers
- [ ] Discovery phase logs each transport scanned and result (including negative)
- [ ] Transport-specific error messages include actionable recovery hints
- [ ] Instrument addresses use a canonical format consistent across all log lines

---

## 15. USER_ACTION — User Interaction & Audit Logging

> §9.2 only gives a 4-line template, insufficient for multi-user systems, compliance audit, and config-change traceability. This section elevates "user interaction" to a first-class specification.

### 15.1 Field Dictionary (MANDATORY)

All user interaction events use the following field order for `grep` / `awk` / log parsers:

```
USER_ACTION | ts=ISO8601 | user=<uid> | role=<role> | session=<sid> |
            tenant=<tid> | action=<verb> | resource=<id> |
            result=SUCCESS|DENIED|FAIL | reason=<why>
```

| Field | Required | Description |
|-------|----------|-------------|
| `user`    | ✓ | **Internal non-reversible ID** (not email, phone, real name) |
| `role`    | - | `admin` / `operator` / `guest` / custom role |
| `session` | - | Session ID, associated with §7.1 `req=` |
| `tenant`  | - | Tenant ID for multi-tenant systems; omit for single-tenant |
| `action`  | ✓ | Action verb (see §15.2-§15.7 catalog) |
| `resource`| - | Target resource ID |
| `result`  | ✓ | `SUCCESS` / `DENIED` / `FAIL` |
| `reason`  | - | Failure reason (wrong password, insufficient role, validation error) |

### 15.2 Login Lifecycle

```python
audit = logging.getLogger("audit.user")

# Login success
audit.info("USER_ACTION | user=%s | role=%s | session=%s | action=LOGIN | result=SUCCESS | src=%s",
           user_id, role, session_id, source_ip)

# Login failure (wrong password / unknown account)
audit.warning("USER_ACTION | user=%s | action=LOGIN | result=FAIL | reason=%s | src=%s",
              user_id, "invalid_password", source_ip)

# Account locked (consecutive failures)
audit.critical("USER_ACTION | user=%s | action=LOGIN | result=DENIED | reason=account_locked | attempts=%d",
               user_id, failed_attempts)

# Logout
audit.info("USER_ACTION | user=%s | session=%s | action=LOGOUT | result=SUCCESS",
           user_id, session_id)

# Session timeout
audit.info("USER_ACTION | user=%s | session=%s | action=SESSION_TIMEOUT | duration=%ds",
           user_id, session_id, session_duration_sec)
```

### 15.3 Config Change Audit (Old → New)

```python
# Normal config change
audit.info("USER_ACTION | user=%s | session=%s | action=CONFIG_SET | resource=%s | "
           "key=%s | old=%s | new=%s | result=SUCCESS",
           user_id, session_id, service_id, key, old_val, new_val)

# Sensitive credentials: write [REDACTED] instead of plaintext
audit.info("USER_ACTION | user=%s | action=CONFIG_SET | key=api_token | "
           "old=[REDACTED] | new=[REDACTED] | result=SUCCESS", user_id)
```

### 15.4 User Cancellation

Distinguishing "user-initiated stop" from "system crash" is operationally critical:

```python
audit.info("USER_ACTION | user=%s | session=%s | action=CANCEL | resource=%s | "
           "reason=user_requested | progress=%.0f%% | elapsed=%.1fs",
           user_id, session_id, job_id, progress_pct, elapsed_s)
```

### 15.5 Input Validation Failure

Values MUST be `[REDACTED]` to avoid leaking PII:

```python
audit.warning("USER_ACTION | user=%s | action=INPUT_VALIDATE | resource=%s | "
              "field=%s | reason=%s | value=[REDACTED] | result=FAIL",
              user_id, form_id, field_name, validation_error)
```

### 15.6 Permission Denied

```python
audit.error("USER_ACTION | user=%s | role=%s | action=%s | resource=%s | "
            "required_role=%s | result=DENIED | reason=insufficient_role",
            user_id, role, attempted_action, resource_id, required_role)
```

### 15.7 Data-Sensitive Operations (Export / Delete / Download)

GDPR / SOC2 compliance:

```python
audit.info("USER_ACTION | user=%s | session=%s | action=EXPORT_DATA | "
           "resource=%s | rows=%d | format=%s | confirmation=2FA | result=SUCCESS",
           user_id, session_id, dataset_id, row_count, export_format)
```

### 15.8 §15 Checklist

- [ ] All user-triggered events include `user_id` (non-reversible ID, not email/phone)
- [ ] Login/logout/auth failures logged in separate `audit.user` child logger
- [ ] Config changes record `old=` → `new=` (sensitive fields `[REDACTED]`)
- [ ] User cancellation (INFO) vs system crash (CRITICAL) strictly distinguished
- [ ] Input validation failures record field name + reason, value `[REDACTED]`
- [ ] Permission denials include `user_id` + `required_role` + `resource`
- [ ] Data export/delete/download records `rows=` / `format=` / `confirmation=2FA`
- [ ] `audit` logger is never disabled (INFO minimum in all environments)

---

## 16. HEALTH & METRICS — System Resources & Periodic Indicators

> §4.2 only shows a demo HEALTH line. For 2024+ ML/AI agent work it is severely insufficient: GPU/VRAM = 0 coverage; disk/FDS/network/coroutines/cgroup = 0; threshold table, collection libraries, cross-platform strategy all missing.

### 16.1 Required vs Recommended Metrics

**Required (any long-running process must include):**

| Field | Meaning | Note |
|-------|---------|------|
| `uptime` | Process uptime in seconds | Mark `final=true` on shutdown |
| `rss` | Process RSS (MB) + `Δ` since last HEALTH | |
| `cpu` | Process CPU % | |
| `thr` | Thread count | |
| `q` | Business queue current/max | |

**Recommended (use by process type):**

| Field | When to include | Source |
|-------|-----------------|--------|
| `gpu=<idx>:util=\|mem=\|temp=\|pwr=` | ML training / inference | `pynvml` / `torch.cuda` |
| `disk:<mount>=<pct>%` | Disk-writing daemons | `psutil.disk_usage` |
| `fds=<used>/<max>` | High-concurrency services | `psutil.Process.num_fds` |
| `net:est=\|tw=\|cw=` | API server / message consumers | `psutil.net_connections` |
| `load=<avg1>` | CPU scheduling diagnosis | `psutil.getloadavg` |
| `coro=<n>` | asyncio projects | `asyncio.all_tasks` |

### 16.2 Standard HEALTH Line Format

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

**Multi-GPU**: concatenate as `gpu=0:...|gpu=1:...`. Never report only the first GPU.

### 16.3 Recommended Collection Libraries

| Library | Purpose | Platform |
|---------|---------|----------|
| `psutil` | Cross-platform process/system metrics | Linux / macOS / Windows |
| `pynvml` | NVIDIA GPU metrics (util/mem/temp/pwr) | When NVIDIA driver present |
| `torch.cuda` | Training-framework VRAM allocation | PyTorch projects |
| `GPUtil` | High-level wrapper around pynvml | Same as pynvml |

macOS has no `/proc`; Windows requires `wmi` / `psutil`. Apple Silicon (MPS) VRAM statistics need `torch.mps` / Metal API — **no unified library** yet. If writing macOS ML tools, document the limitation in the README.

### 16.4 Threshold Alert Table (defaults, adjustable)

| Metric | WARNING | ERROR | CRITICAL |
|--------|---------|-------|----------|
| RSS growth rate | > 50 MB/h | > 200 MB/h | monotonically rising 1h |
| Disk usage | > 80% | > 90% | > 95% or inode > 90% |
| VRAM usage | > 85% | > 95% | = 100% (OOM imminent) |
| GPU temperature | > 80°C | > 88°C | > 95°C (throttle/shutdown) |
| FDs usage | > 70% ulimit | > 85% ulimit | = ulimit (connection failure) |
| CLOSE_WAIT | > 100 | > 1000 | > 10000 (connection leak) |
| Load average | > cores × 1.5 | > cores × 3 | > cores × 5 |

### 16.5 Collection Template (psutil + pynvml)

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
    """Collect one round of metrics. Sub-component failures MUST NOT raise."""
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

### 16.6 Periodic Output Loop

```python
HEALTH_INTERVAL = 30  # seconds
_prev = None
while not shutdown.is_set():
    h = collect_health(prev_rss_mb=_prev)
    _prev = h["rss_mb"]
    log.info(format_health(h))
    shutdown.wait(HEALTH_INTERVAL)

# Last line on shutdown
log.info("HEALTH | final=true | " + format_health(collect_health(_prev)).removeprefix("HEALTH | "))
```

### 16.7 Short-Task Strategy

When task duration < 2 × `HEALTH_INTERVAL`, **do NOT emit HEALTH** — use a START/STOP summary instead:

```python
# < 60s batch: start log + end summary suffices
log.info("BATCH started: rows=%d input=%s", row_count, input_path)
result = process(input_path)
log.info("BATCH completed: rows=%d dur=%.2fs ok=%d fail=%d",
         row_count, elapsed, ok_count, fail_count)
```

### 16.8 Container / cgroup Caveats

- In containers, `RSS` may be truncated by cgroup limit (`memory.limit_in_bytes`)
- K8s deployments should additionally capture `cgroup_mem_used / cgroup_mem_max` to avoid misjudgment
- `cgroup_cpu_quota` and `cpu.shares` should be logged at INFO on startup (informs CPU % interpretation)

### 16.9 §16 Checklist

- [ ] HEALTH line includes the five required fields: `uptime / rss / cpu / thr / q`
- [ ] GPU workloads record `gpu=idx:util|mem|temp|pwr`
- [ ] High-concurrency services add `fds` + `net:est|tw|cw`
- [ ] Disk-writing daemons add `disk:mount=pct%`
- [ ] First HEALTH ≥ 5s after start; `final=true` on shutdown
- [ ] Threshold alerts per §16.4 table
- [ ] Use `psutil` + `pynvml` (NVIDIA) or document macOS/AMD limits in README
- [ ] Container deployments additionally record cgroup limits
- [ ] Short tasks (< 2× interval) skip HEALTH, use START/STOP summary

---

## 17. Physical Interface Layer — General-Purpose Physical Interface Logging

> §14 covers **measurement instruments only** (USB-TMC/VISA/TCP-VXI-11/Serial/GPIB). This section extends to general physical interfaces: embedded MCUs, Raspberry Pi, industrial PLCs, machine vision, motion control.

### 17.1 Extended Transport Tags

Reusing §14.1's `[transport=xxx][addr=xxx]` format, additional tags:

| Layer | Tag | Address format | Typical use |
|-------|-----|----------------|-------------|
| GPIO | `transport=gpio` | `chip=0/line=17` or `PIN17` | Raspberry Pi, MCU, PLC DO |
| I²C | `transport=i2c` | `/dev/i2c-1@0x48` | sensors, EEPROM, ADC |
| SPI | `transport=spi` | `/dev/spidev0.0@1MHz` | FPGA, Flash, high-speed ADC |
| MCU UART | `transport=uart` | `USART2@115200` / `ttyAMA0@9600` | MCU debug, TTL serial |
| CAN / CAN-FD | `transport=can` | `can0@1Mbps` / `can0@500kbps-fd` | automotive ECU, industrial fieldbus |
| Modbus RTU/TCP | `transport=modbus` | `slave=0x01@COM3@9600` / `tcp:host:502/slave=0x01` | PLC, VFD, meter |
| OPC-UA | `transport=opcua` | `opc.tcp://host:4840/ns=2;s=Channel1` | SCADA, Industry 4.0 |
| USB camera (UVC) | `transport=uvc` | `/dev/video0@MJPEG@1920x1080@30fps` | webcam, machine vision |
| CSI camera | `transport=csi` | `csi0@imx477@4056x3040` | Raspberry Pi camera |
| GigE Vision | `transport=gige` | `192.168.1.50@GVSP@Basler-acA1920` | industrial camera |
| Step/servo pulse | `transport=pulse` | `AXIS0@dir=DIR1,pul=PUL2,mode=CW/CCW` | stepper, servo |
| PWM | `transport=pwm` | `chip=0/channel=0@1kHz@50%` | servo, dimming |
| Encoder | `transport=qenc` | `chip=0/quad=0` | position feedback |
| BLE | `transport=ble` | `AA:BB:CC:DD:EE:FF@GATT` | BLE |
| LoRa | `transport=lora` | `sx1276@868MHz@SF7/BW125` | long-range low-power |
| PCIe DAQ | `transport=pcie` | `NI-PXI-6366/slot=2/ai0` | data acquisition |

**Address format rules:**
- Linux device paths preferred: `/dev/i2c-1`, `/dev/spidev0.0`, `/dev/video0`, `can0`
- Physical parameters explicitly encoded: `@0x48` (I²C address), `@1Mbps` (CAN rate), `@1920x1080@30fps` (camera resolution/fps)
- Multi-contributor projects MUST maintain an "address dictionary" in the README

### 17.2 Bus Enumeration & Discovery

At startup, **MUST enumerate** all buses and devices used, **including negative results**:

```python
logger.info("[discover][i2c] Scanning /dev/i2c-1 for slave devices...")
logger.info("[discover][i2c] Found 3 devices: 0x48 (TMP102), 0x68 (MPU6050), 0x77 (BMP280)")
logger.warning("[discover][can] can0 interface not up (check: ip link set can0 up?)")
logger.error("[discover][uvc] /dev/video0 missing (camera not plugged in?)")
logger.info("[discover][gpio] GPIO17 configured as OUTPUT (default LOW)")
logger.warning("[discover][gpio] GPIO4 already in use (used by: gpio_led.py), skipped this run")
```

### 17.3 Connection Lifecycle (extends §14.2)

```python
# I²C device init
logger.info("[transport=i2c][addr=/dev/i2c-1@0x48] Initialized: chip=MPU6050, who_am_i=0x68")

# CAN channel up
logger.info("[transport=can][addr=can0@1Mbps] up: ip link set can0 up type can bitrate 1000000")

# Camera open
logger.info("[transport=uvc][addr=/dev/video0@MJPEG@1920x1080@30fps] Opened: idn='Logitech BRIO, USB 3.0'")

# Unexpected disconnect (CAN bus-off)
logger.warning("[transport=can][addr=can0@1Mbps] Entered Bus-Off, recovery in 5s...")
```

### 17.4 Atomic-Operation Logging Granularity (KEY ADDITION)

Physical interface calls are often **high-frequency and atomic** — granularity rules differ from business logs:

| Frequency | Recommended granularity | Example |
|-----------|-------------------------|---------|
| < 10 Hz | Every event at INFO / DEBUG | GPIO LED toggle, relay control |
| 10 - 100 Hz | START/STOP at INFO, sample every 100 at DEBUG | UART text command, Modbus polling |
| 100 - 1000 Hz | Only errors at WARNING/ERROR; START/STOP + periodic HEALTH summary | I²C sensor sampling, SPI ADC scan |
| > 1000 Hz | **NEVER log per-event**; child logger + periodic counter + on-anomaly | CAN 1Mbps bus load, SPI DMA burst |

**Anti-pattern** (per-event logging on 1 kHz CAN bus causes log explosion):

```python
# BAD: 1000 lines/s → log file overflow → IO blocks → actual frame loss
for msg in can_bus:
    logger.info("[transport=can][addr=can0@1Mbps] recv 0x%03X: %s",
                msg.arbitration_id, msg.data.hex())
```

**Correct pattern** (child logger + periodic summary + per-event on errors):

```python
can_log = logging.getLogger("transport.can")
frame_count = err_count = 0
last_summary = time.monotonic()
for msg in can_bus:
    frame_count += 1
    if msg.is_error_frame:
        can_log.warning("[transport=can][addr=can0@1Mbps] error frame: id=0x%03X, err=%s",
                        msg.arbitration_id, msg.error)
        err_count += 1
    elif msg.arbitration_id in CRITICAL_IDS:
        can_log.info("[transport=can][addr=can0@1Mbps] critical frame: id=0x%03X, data=%s",
                     msg.arbitration_id, msg.data.hex())
    if time.monotonic() - last_summary >= 1.0:
        can_log.debug("[transport=can][addr=can0@1Mbps] 1s summary: frames=%d, err=%d, busload=%.1f%%",
                      frame_count, err_count, busload_pct)
        frame_count = err_count = 0
        last_summary = time.monotonic()
```

### 17.5 Embedded Buses (GPIO / I²C / SPI / UART)

```python
# GPIO
logger.debug("[transport=gpio][addr=chip=0/line=17] set HIGH (mode=OUTPUT, pull=UP)")
logger.warning("[transport=gpio][addr=chip=0/line=4] pin conflict: in use by gpio_led.py, this write skipped")
logger.error("[transport=gpio][addr=chip=0/line=27] read failed: gpiod_line_request_get failed (ENODEV)")

# I²C
logger.debug("[transport=i2c][addr=/dev/i2c-1@0x48] reg_read reg=0x00 len=2 -> 0x%04x", value)
logger.warning("[transport=i2c][addr=/dev/i2c-1@0x48] NACK retry: 3/3 still failing, check wiring/pull-up")
logger.error("[transport=i2c][addr=/dev/i2c-1@0x68] bus error: bus=-EBUSY (other master holding?)")

# SPI
logger.debug("[transport=spi][addr=/dev/spidev0.0@1MHz] xfer len=%d mode=%d cs_high=%s",
             len(tx), mode, cs_active)
logger.warning("[transport=spi][addr=/dev/spidev0.0@1MHz] clock division: requested 1MHz, actual 1.024MHz (HW limit)")
```

### 17.6 Industrial / Automotive Buses (CAN / Modbus / OPC-UA)

```python
# CAN
logger.info("[transport=can][addr=can0@1Mbps] send id=0x18FEF100 DLC=8 data=%s", data.hex())
logger.warning("[transport=can][addr=can0@1Mbps] arbitration lost: id=0x%03X after %d retries", can_id, retries)
logger.error("[transport=can][addr=can0@1Mbps] Bus-Off: TEC=%d REC=%d, entering recovery", tec, rec)

# Modbus
logger.debug("[transport=modbus][addr=tcp:192.168.1.10:502/slave=0x01] fc=0x03 addr=40001 len=10 -> %s",
             regs.hex())
logger.warning("[transport=modbus][addr=slave=0x01@COM3@9600] CRC error: expected 0x%04x, got 0x%04x (EMI?)",
               expected, actual)
logger.error("[transport=modbus][addr=tcp:192.168.1.10:502/slave=0x01] slave no response: fc=0x06 write 40001, 3s timeout")

# OPC-UA
logger.info("[transport=opcua][addr=opc.tcp://plc.local:4840] Connected: server='Siemens S7-1500', endpoint=Signed")
logger.debug("[transport=opcua][addr=ns=2;s=Channel1.Temperature] read: value=%.2f °C, status=Good", value)
```

### 17.7 Machine Vision (UVC / CSI / GigE Vision)

```python
logger.info("[transport=uvc][addr=/dev/video0@MJPEG@1920x1080@30fps] Opened: backend=V4L2, format=MJPG")
logger.warning("[transport=uvc][addr=/dev/video0] framerate degraded: requested 30fps, actual %.1f fps (USB bandwidth?)", actual_fps)
logger.error("[transport=uvc][addr=/dev/video0] select timeout: no camera data (USB unplugged? driver crash?)")

logger.info("[transport=gige][addr=192.168.1.50@GVSP@Basler-acA1920] Connected: model='acA1920-155um', serial=40001234, MTU=9000")
logger.warning("[transport=gige][addr=192.168.1.50] high packet loss: %.2f%% (check: cable / jumbo frames / flow control)", loss_pct)
```

### 17.8 Motion Control (Stepper / Servo / Encoder / PWM)

```python
# Stepper
logger.info("[transport=pulse][addr=AXIS0@dir=DIR1,pul=PUL2] motion started: target=%d pulses @ %.0f Hz",
            target_pulses, step_freq)
logger.warning("[transport=pulse][addr=AXIS0] step loss detected: encoder feedback %d, commanded %d, error %d (>= threshold)",
               actual, commanded, abs(actual - commanded))

# Encoder
logger.debug("[transport=qenc][addr=chip=0/quad=0] periodic read: count=%d, velocity=%.2f pulse/s", count, vel)

# PWM servo
logger.debug("[transport=pwm][addr=chip=0/channel=0@50Hz] pulse width: %.0f µs (%.1f°)", pulse_us, angle)
```

### 17.9 §17 Checklist

- [ ] Every physical interface log carries `[transport=xxx][addr=xxx]` tag (see §17.1)
- [ ] Address format is codebase-consistent (Linux device path + physical params); maintain an address dictionary in README
- [ ] Startup records full bus/device enumeration, including negatives (missing device, bus-off, unauthorized)
- [ ] Connect/enable/open at INFO; unexpected disconnect/error at WARNING or ERROR
- [ ] High-frequency physical interfaces (>100 Hz) NEVER log per-event; use child logger + periodic summary + per-event on errors
- [ ] Atomic operations (pin toggle, register read/write, single CAN frame) at DEBUG; batch operations at INFO
- [ ] HEALTH line (§16) extended with per-bus key metrics (I²C NACK count, CAN Bus-Off count, frame loss rate)
- [ ] Protocol/physical layer separated: raw bytes in `transport.xxx` child logger; business logic in main logger
- [ ] Error messages include actionable recovery hints ("check wiring", "check pull-up resistor", "ip link set can0 up")
- [ ] Physical parameter changes (clock rate, CAN baud, sample rate) logged at INFO

---

## 18. FILE_IO — File & Storage I/O Logging

> Only §4.4 mentions "file close failure" in the entire spec. Config files, data files, atomic writes, chunked reads, handle leaks, and failure classification are all unaddressed. Agents writing file I/O code have no rule to follow.

### 18.1 Unified `FILE_IO` Prefix (MANDATORY)

All business file I/O logs MUST begin with `FILE_IO | op=<op> kind=<kind> path=<path>`:

```python
logger.info("FILE_IO | op=open   kind=config path=/etc/myapp/config.yaml size=2.3KB")
logger.info("FILE_IO | op=read   kind=config path=/etc/myapp/config.yaml bytes=2341")
logger.info("FILE_IO | op=write  kind=data   path=/data/run_001.csv rows=100000 bytes=12.4MB dur=3.21s")
logger.info("FILE_IO | op=close  kind=data   path=/data/run_001.csv rows=100000 bytes=12.4MB")
logger.info("FILE_IO | op=rename kind=tmp     path=/tmp/x.tmp -> /data/run_001.csv")
logger.info("FILE_IO | op=delete kind=tmp     path=/tmp/x.tmp reason=committed")
```

| Field | Values |
|-------|--------|
| `op`   | `open` / `read` / `write` / `flush` / `fsync` / `close` / `rename` / `delete` / `lock` / `unlock` |
| `kind` | `config` / `data` / `tmp` / `state` / `lock` / `log` |
| `path` | Absolute path preferred |
| `bytes` | With unit (`B`/`KB`/`MB`/`GB`) |
| `rows` | Row count (when applicable) |
| `dur`  | Duration with unit (`ms`/`s`) |

### 18.2 File Classification `kind`

| kind | Meaning | Typical |
|------|---------|---------|
| `config` | Config files | YAML / JSON / TOML / INI / `.env` |
| `data` | Business data | CSV / Parquet / HDF5 / NumPy / pickle / protobuf |
| `tmp` | Temp files | `tempfile.NamedTemporaryFile` / `*.tmp` |
| `state` | State / snapshot | checkpoint / heartbeat / lock file |
| `lock` | Lock files | `*.lock` / `flock` handle |
| `log` | Log files themselves | governed by §3.1 / §3.2 |

### 18.3 Lifecycle Logging Pattern

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

# ── 2. read / write (aggregate by batch) ────────
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

# ── 3. flush + fsync (strong durability) ────────
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

**Rules:**
- `open` failure MUST distinguish `FileNotFoundError` / `PermissionError` / other `OSError`
- `flush` / `fsync` failure MUST escalate to **CRITICAL** (data may be lost)
- Write loops **MUST NOT** log per event; use `rows % BATCH == 0` modulo sampling

### 18.4 Atomic Write Pattern (`tmp + os.replace`)

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

**Rules:**
- Any "business data" write to disk MUST go through atomic-write
- `mkstemp` in target directory (ensures `rename` does not cross filesystems)
- `fsync` failure MUST NOT be swallowed; log CRITICAL
- `os.replace` failure MUST log ERROR and clean up tmp

### 18.5 Config File Loading

Loading config at startup is a P0 critical path:

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

**Rules:**
- Config load failure = **CRITICAL** (system cannot start)
- Record `sha256` first 12 chars for version comparison
- Record `keys=` / `sections=` count for post-hoc verification

### 18.6 Large File Chunked I/O

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

**Rules:**
- Single `read()` MUST NOT exceed 64 MiB
- No logs inside the loop; single summary line after the loop
- Throughput unit `MB/s` (decimal) or `MiB/s` (binary) MUST be explicit

### 18.7 Failure Mode → Level Decision Table

| Exception / scenario | Level | Example message |
|----------------------|-------|-----------------|
| `FileNotFoundError` (read data) | ERROR | `op=open FAILED reason=not_found` |
| `FileNotFoundError` (write data) | CRITICAL | Path must exist; should not be missing |
| `PermissionError` | CRITICAL | `op=open FAILED reason=permission_denied user=...` |
| `IsADirectoryError` | ERROR | `op=open FAILED reason=is_directory` |
| `OSError(errno=28)` ENOSPC | CRITICAL | `op=write FAILED reason=disk_full free=0` |
| `OSError(errno=122)` EDQUOT | CRITICAL | quota exhausted |
| `UnicodeDecodeError` | ERROR | `op=read FAILED reason=encoding pos=%d` |
| `flush` / `fsync` failure | CRITICAL | `op=fsync FAILED durability=NOT_GUARANTEED` |
| `close` failure | ERROR | `op=close FAILED (handle leak suspected)` |
| `os.replace` failure | ERROR | `op=rename FAILED (cleanup attempted)` |
| Temp file residual | WARNING | `op=delete FAILED reason=residual` |
| File lock acquisition failure | WARNING | `op=lock FAILED timeout=5s` |

### 18.8 Handle Leak Detection

Add file handle metrics to the §16 HEALTH line:

```python
log.info("HEALTH | ... | fds=%d open_files=%d",
         fds_used, n_open_files)
```

**Rules:**
- `open_files` monotonically rising → WARNING
- `open_files > 80%` of system limit → CRITICAL
- On process exit, every open file MUST be removable from `lsof` output

### 18.9 §18 Checklist

- [ ] File I/O logs begin with `FILE_IO | op=<op> kind=<kind> path=<path>`
- [ ] `kind` is one of the six categories in §18.2
- [ ] Every file path appears at least once (absolute path preferred)
- [ ] File size and duration include units, using `bytes=` / `dur=` fields
- [ ] Business data writes go through `tmp + os.replace` atomic write
- [ ] `fsync` failure escalates to CRITICAL
- [ ] Write loops aggregate logs with `rows % BATCH == 0`
- [ ] Large files read in chunks ≤ 64 MiB
- [ ] Config load failure = CRITICAL
- [ ] HEALTH line (§16) includes `open_files` handle metric
- [ ] Temp file residuals generate ERROR on cleanup failure
- [ ] Encoding errors (`UnicodeDecodeError`) handled in a dedicated branch

---

**End of Agent Logging Standard**

*Generated from OptoSync project analysis (2026-06-01); extended §15-§18 (USER_ACTION / HEALTH & METRICS / Physical Interface / FILE_IO) on 2026-06-21.*


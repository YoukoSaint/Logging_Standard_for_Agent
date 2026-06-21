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

### Core setup (§3)

- [ ] `setup_logging()` called in entry point with timestamped file
- [ ] File handler = DEBUG, console handler = INFO
- [ ] Every module uses `logging.getLogger(__name__)`
- [ ] Consistent prefixes for grep (`HEALTH |`, `USER_ACTION |`, `FILE_IO |`, `[transport=xxx][addr=xxx]`)

### Logging content (§4 / §6 / §8)

- [ ] START/STOP logged for long-running operations with metrics
- [ ] `logger.exception()` used in except blocks for unexpected errors
- [ ] All quantitative logs include units (Hz, MB, ms, %, °C, W)

### Performance & security (§5 / §9)

- [ ] No log calls inside >100 Hz loops
- [ ] `%s` formatting used (not f-strings)
- [ ] No passwords, tokens, or PII in any log

### User interaction (§15)

- [ ] All user-triggered events include `user_id` (non-reversible ID)
- [ ] Login/logout/auth failures logged in separate `audit.user` logger
- [ ] Config changes record `old=` → `new=` (sensitive fields `[REDACTED]`)
- [ ] User cancellation (INFO) vs system crash (CRITICAL) strictly distinguished
- [ ] Permission denials include `user_id` + `required_role` + `resource`

### System resources & periodic metrics (§16)

- [ ] Health monitoring added for daemon processes (HEALTH line)
- [ ] HEALTH line includes the five required fields: `uptime / rss / cpu / thr / q`
- [ ] GPU workloads record `gpu=idx:util|mem|temp|pwr`
- [ ] High-concurrency services add `fds` + `net:est|tw|cw`
- [ ] Disk-writing daemons add `disk:mount=pct%`
- [ ] Threshold alerts per §16.4 table (WARNING/ERROR/CRITICAL)

### Physical interfaces (§14 / §17)

- [ ] Every instrument log line tagged with `[transport=xxx][addr=xxx]`
- [ ] Embedded/industrial/vision interfaces (§17) use corresponding transport tags
- [ ] High-frequency physical interfaces (>100 Hz) use child logger + periodic summary
- [ ] Startup records bus/device enumeration (including negatives)

### File I/O (§18)

- [ ] File I/O logs begin with `FILE_IO | op=<op> kind=<kind> path=<path>`
- [ ] Business data writes go through `tmp + os.replace` atomic write
- [ ] `fsync` failure escalates to CRITICAL
- [ ] Write loops aggregate with `rows % BATCH == 0`
- [ ] Config load failure = CRITICAL

> Full per-dimension checklists: §15.8 / §16.9 / §17.9 / §18.9.



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

## 12. USER_ACTION — User Interaction & Audit (condensed)

> Multi-user systems require `user_id` context on every user-triggered event. See **§15** in AGENT_LOGGING_STANDARD_zh.md for the full specification.

```python
audit = logging.getLogger("audit.user")

# Login lifecycle
audit.info("USER_ACTION | user=%s | role=%s | session=%s | action=LOGIN | result=SUCCESS | src=%s",
           user_id, role, session_id, source_ip)
audit.warning("USER_ACTION | user=%s | action=LOGIN | result=FAIL | reason=%s | src=%s",
              user_id, "invalid_password", source_ip)
audit.critical("USER_ACTION | user=%s | action=LOGIN | result=DENIED | reason=account_locked | attempts=%d",
               user_id, failed_attempts)

# Config change with redaction
audit.info("USER_ACTION | user=%s | action=CONFIG_SET | key=api_token | "
           "old=[REDACTED] | new=[REDACTED] | result=SUCCESS", user_id)

# Permission denial
audit.error("USER_ACTION | user=%s | role=%s | action=%s | resource=%s | "
            "required_role=%s | result=DENIED | reason=insufficient_role",
            user_id, role, attempted_action, resource_id, required_role)
```

**Field dictionary**: `user=<uid> | role=<role> | session=<sid> | tenant=<tid> | action=<verb> | resource=<id> | result=SUCCESS|DENIED|FAIL | reason=<why>`

**Rules:**
- `user` MUST be an internal non-reversible ID (not email/phone/name)
- Sensitive values (config, user input) → `[REDACTED]`
- `audit` logger is never disabled

## 13. HEALTH & METRICS — System Resources (condensed)

> §4.2 only shows a demo line. For ML/AI workloads, GPU/VRAM/disk/FDS/network are MANDATORY. See **§16** for the full spec, threshold table, and `psutil + pynvml` collection template.

```python
log.info(
    "HEALTH | uptime=%ds | rss=%.1fMB (Δ%+.1f) | cpu=%.1f%% (load=%.2f) | "
    "thr=%d coro=%d | fds=%d/%d | net:est=%d|tw=%d|cw=%d | "
    "gpu=%d:util=%d%%|mem=%d/%dMB(%.0f%%)|temp=%dC|pwr=%dW | "
    "disk:root=%.0f%%|data=%.0f%%|log=%.0f%% | q=%d/%d | rate=%.1f/s",
    ...)
```

**Required fields**: `uptime / rss / cpu / thr / q`
**Recommended by process type**: `gpu=...`, `disk:mount=...`, `fds=...`, `net:...`, `load=...`, `coro=...`

**Threshold table (defaults):**

| Metric | WARNING | ERROR | CRITICAL |
|--------|---------|-------|----------|
| RSS growth rate | > 50 MB/h | > 200 MB/h | monotonically rising 1h |
| Disk usage | > 80% | > 90% | > 95% or inode > 90% |
| VRAM usage | > 85% | > 95% | = 100% (OOM imminent) |
| GPU temperature | > 80°C | > 88°C | > 95°C |
| FDs usage | > 70% ulimit | > 85% ulimit | = ulimit |

**Short tasks (< 2 × interval)** skip HEALTH — use START/STOP summary instead.

## 14. Physical Interface Layer — General-Purpose (condensed)

> §14 covers measurement instruments only. §17 extends to general physical interfaces. Tag every line with `[transport=xxx][addr=xxx]`.

| Layer | Tag | Address format |
|-------|-----|----------------|
| GPIO | `transport=gpio` | `chip=0/line=17` |
| I²C | `transport=i2c` | `/dev/i2c-1@0x48` |
| SPI | `transport=spi` | `/dev/spidev0.0@1MHz` |
| CAN | `transport=can` | `can0@1Mbps` |
| Modbus | `transport=modbus` | `slave=0x01@COM3@9600` / `tcp:host:502/slave=0x01` |
| OPC-UA | `transport=opcua` | `opc.tcp://host:4840/ns=2;s=Channel1` |
| UVC | `transport=uvc` | `/dev/video0@MJPEG@1920x1080@30fps` |
| GigE | `transport=gige` | `192.168.1.50@GVSP@Basler-acA1920` |
| Step/servo pulse | `transport=pulse` | `AXIS0@dir=DIR1,pul=PUL2` |
| PWM | `transport=pwm` | `chip=0/channel=0@1kHz@50%` |

**Atomic-operation granularity** (KEY rule — not in §14):

| Frequency | Granularity |
|-----------|-------------|
| < 10 Hz | Every event at INFO/DEBUG |
| 10–100 Hz | START/STOP at INFO, sample every 100 at DEBUG |
| 100–1000 Hz | Only errors at WARNING/ERROR; periodic HEALTH summary |
| > 1000 Hz | **NEVER per-event** — child logger + periodic counter + on-anomaly |

**Discovery MUST log negatives**: `[discover][can] can0 not up`, `[discover][i2c] no devices found`, etc.

## 15. FILE_IO — File & Storage I/O (condensed)

> §4.4 only covers `close` failure. See **§18** for the full spec.

```python
# Unified prefix
logger.info("FILE_IO | op=write kind=data path=/data/run_001.csv rows=100000 bytes=12.4MB dur=3.21s")

# Atomic write (mandatory for business data)
def atomic_write(path, data):
    fd, tmp = tempfile.mkstemp(prefix=".tmp_", dir=os.path.dirname(path))
    try:
        with os.fdopen(fd, "wb") as f:
            f.write(data); f.flush(); os.fsync(f.fileno())
        os.replace(tmp, path)
        logger.info("FILE_IO | op=rename kind=data path=%s -> %s bytes=%d", tmp, path, len(data))
    except Exception:
        logger.exception("FILE_IO | op=write kind=data path=%s FAILED", path)
        try: os.unlink(tmp)
        except FileNotFoundError: pass
        raise
```

**Field dictionary**: `op=<op> kind=<kind> path=<path> [bytes=...] [rows=...] [dur=...]`
**`op`**: `open` / `read` / `write` / `flush` / `fsync` / `close` / `rename` / `delete` / `lock` / `unlock`
**`kind`**: `config` / `data` / `tmp` / `state` / `lock` / `log`

**Failure → level**:
- `FileNotFoundError` (read) → ERROR
- `FileNotFoundError` (write) → CRITICAL
- `PermissionError` → CRITICAL
- `OSError(errno=28)` ENOSPC → CRITICAL
- `flush` / `fsync` failure → CRITICAL
- `close` failure → ERROR with `logger.exception`

**Chunked reads**: `read()` MUST NOT exceed 64 MiB; no logs inside the loop.

## Anti-Patterns Quick Reference

| ❌ Wrong | ✅ Right |
|----------|----------|
| `print("starting...")` | `logger.info("starting...")` |
| `logger.error("Failed")` | `logger.error("Device %s: %s", name, err)` |
| `logger.debug(f"{x}")` | `logger.debug("%s", x)` |
| `except: pass` | `except Exception: logger.exception(...)` |
| `logger.info("pwd=%s", pwd)` | `logger.info("user=%s auth=OK", user)` |
| `logger.error("Permission denied")` | `logger.error("AUTHZ_DENIED \| user=%s \| resource=%s", uid, rid)` |
| `for msg in can_bus: logger.info(...)` | child logger + periodic summary (§14) |
| `data = open("big.bin","rb").read()` | chunked ≤ 64 MiB (§15) |
| `open("/data/x.csv","w").write(...)` | `tmp + os.replace` (§15) |
| `HEALTH \| rss=... cpu=...` (ML workload) | include `gpu=idx:util\|mem\|temp\|pwr` (§13) |

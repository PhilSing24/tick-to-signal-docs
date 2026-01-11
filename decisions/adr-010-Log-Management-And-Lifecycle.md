# ADR-010: Log Management and Lifecycle

## Status
Accepted (Updated 2026-01-09)

## Date
Original: 2026-01-07
Updated: 2026-01-09

## Context

The system uses tickerplant (TP) binary logs for durability and recovery (see ADR-003, ADR-006):
- Single log file per day: `logs/YYYY.MM.DD.log`
- Contains all market data (trades + quotes) in chronological order

As the system runs, log files accumulate:
- ~70 MB per hour (at current message rates)
- ~1.7 GB/day if running 24 hours

Without management:
- Disk space exhaustion
- No visibility into log health
- Manual cleanup required

A decision is required on how log files are managed throughout their lifecycle.

## Notation

| Acronym | Definition |
|---------|------------|
| LOG | Log Manager Process |
| RDB | Real-Time Database |
| RTE | Real-Time Engine |
| TP | Tickerplant |

## Decision

A dedicated **LOG manager process** (port 5014) handles log lifecycle management.

### Architecture

```
                    TP :5010
                       |
                       v
              logs/YYYY.MM.DD.log
             (trades + quotes)
                       |
                       v
    +------------------+------------------+
    |                  |                  |
    v                  v                  v
RDB :5011         RTE :5012          LOG :5014
(replay)          (replay)           (management)
```

### LOG Process Responsibilities

| Function | Description |
|----------|-------------|
| Discovery | Find and list all log files |
| Diagnostics | Report sizes, chunk counts, dates |
| Verification | Check log file integrity |
| Cleanup | Delete logs older than retention period |
| Monitoring | Provide log health metrics |

### Process Configuration

```q
.log.cfg.port:5014;              / LOG process port
.log.cfg.logDir:"logs";          / Log directory
.log.cfg.retentionDays:7;        / Default retention period
.log.cfg.cleanupHour:0;          / Cleanup hour (midnight)
```

### Query Interface

**Discovery:**
```q
.log.list[]
/ Returns table:
/ date       file                     sizeMB   chunks
/ --------------------------------------------------------
/ 2026.01.06 `:logs/2026.01.06.log    141.97   359740
/ 2026.01.07 `:logs/2026.01.07.log    70.21    366586
/ 2026.01.08 `:logs/2026.01.08.log    68.45    358921
```

**Summary:**
```q
.log.summary[]
/ Returns table:
/ chunks   sizeMB
/ ----------------
/ 359740   141.97
/ 366586   70.21
/ 358921   68.45
```

**Verification:**
```q
.log.verifyDate[.z.D]
/ LOG: Verifying :logs/2026.01.09.log
/   Chunks: 366586
/   Size: 70891050 bytes
/   Valid: 1
/ Returns:
/ status| `ok
/ file  | `:logs/2026.01.09.log
/ chunks| 366586
/ size  | 70891050
/ valid | 1b

.log.verify[`:logs/2026.01.07.log]
/ LOG: Verifying :logs/2026.01.07.log
/   Chunks: 366586
/   Size: 70891050 bytes
/   Valid: 1
```

**Cleanup:**
```q
.log.cleanup[7]    / Delete logs older than 7 days
.log.cleanup[]     / Use default retention (7 days)
/ LOG: Cleaning up logs older than 2026.01.02 (7 day retention)
/ LOG: Deleting 1 files...
/   Deleting: :logs/2025.12.30.log (2025.12.30)
/ LOG: Cleanup complete - deleted 1 files
```

### Log File Information

**`.log.info[file]`** returns:
```q
(chunks; size; valid)
/ chunks - number of messages in log
/ size   - file size in bytes
/ valid  - 1b if file passes integrity check
```

Uses `-11!(-2;file)` to get chunk count without full replay.

### Retention Policy

| Parameter | Default | Description |
|-----------|---------|-------------|
| `retentionDays` | 7 | Keep logs for 7 days |
| `cleanupHour` | 0 | Run cleanup at midnight (if automated) |

**Cleanup logic:**
1. Calculate cutoff date: `today - retentionDays`
2. List all log files
3. Delete files with date < cutoff
4. Log each deletion

### Automated Cleanup (Optional)

Timer-based cleanup can be enabled:
```q
/ Enable automatic cleanup at configured hour
.z.ts:{.log.checkCleanup[]};
system "t 60000";   / Check every minute
```

**Default:** Disabled. Cleanup is manual via `.log.cleanup[]`.

### Integration with Control Process

CTL (port 5000) includes LOG in health checks:
```q
.ctl.healthCheck[]
/ tp    | 1
/ rdb   | 1
/ rte   | 1
/ tel   | 1
/ logmgr| 1

.ctl.statusTable[]
/ process port status started
/ -------------------------------------------------
/ TP      5010 Live   2026.01.07D03:46:37
/ RDB     5011 Live   2026.01.07D03:46:38
/ RTE     5012 Live   2026.01.07D03:46:39
/ TEL     5013 Live   2026.01.07D03:46:40
/ LOG     5014 Live   2026.01.07D03:46:41
```

### Startup Scripts

LOG is included in all startup scripts:
- `start.sh` - tmux window for LOG
- `start_bg.sh` - background process with PID file
- `stop.sh` - graceful shutdown (SIGTERM then SIGKILL)

## Rationale

This approach was selected because:

**Dedicated process:**
- Separates log management from data processing
- No impact on TP, RDB, or RTE performance
- Can run diagnostics without affecting live system
- Independent restart capability

**Query-based interface:**
- Flexible (any retention period, any date)
- Scriptable (can be called from external tools)
- Transparent (shows what will be deleted)
- Safe (requires explicit call, no silent deletions)

**Lightweight design:**
- No state to maintain (queries files directly)
- Minimal memory footprint
- Simple implementation

## Alternatives Considered

### 1. Log management in TP
Rejected:
- TP should focus on sequencing and publishing
- Adds complexity to critical path
- Risk of affecting ingest performance

### 2. Log management in RDB
Rejected:
- RDB should focus on storage and queries
- Log management is not RDB's responsibility
- Separate process is cleaner

### 3. External tool (bash script)
Rejected:
- Cannot query log internals (chunk counts)
- Cannot integrate with q ecosystem
- Less visibility into log health

### 4. Automatic cleanup only (no manual control)
Rejected:
- Less flexible
- Harder to debug
- Risk of unexpected deletions
- Manual control is safer

### 5. Log compression
Deferred:
- Adds complexity
- Current disk usage acceptable
- Can be added if storage becomes concern

### 6. Log archival to HDB
Deferred:
- No HDB implemented yet
- Out of scope for current phase
- Can be added when HDB is built

### 7. Separate log files per table
Rejected (2026-01-09):
- Breaks chronological order during replay
- Complicates point-in-time recovery
- Single file is simpler to manage
- See ADR-003 for full rationale

## Consequences

### Positive

- Clear visibility into log file status
- Controlled disk space usage
- Easy diagnostics and verification
- Flexible retention policies
- Integration with control process
- No impact on data processing components
- Safe (manual cleanup by default)
- Simple file discovery (one file per day)

### Negative / Trade-offs

- Additional process to manage (port 5014)
- Manual cleanup required (automatic disabled by default)
- No log compression (larger disk usage)
- No archival (logs deleted, not preserved)

These trade-offs are acceptable for the current phase.

## Future Evolution

| Enhancement | Trigger |
|-------------|---------|
| Automatic scheduled cleanup | Production deployment |
| Log compression | Disk space concerns |
| Log archival to HDB | HDB implementation |
| Log streaming to external storage | Cloud deployment |
| Alerting on log corruption | Production monitoring |

## Implementation Files

| File | Purpose |
|------|---------|
| `kdb/logmgr.q` | LOG manager process |
| `kdb/ctl.q` | Control process (includes LOG) |
| `start.sh` | tmux startup (includes LOG) |
| `start_bg.sh` | Background startup (includes LOG) |
| `stop.sh` | Graceful shutdown (includes LOG) |

## Links / References

- `adr-003-tickerplant-logging-and-durability-strategy.md` (log file format, single file)
- `adr-006-recovery-and-replay-strategy.md` (replay mechanism)
- `kdb/logmgr.q` (implementation)

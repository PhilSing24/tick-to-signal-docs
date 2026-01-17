# ADR-010: Log Management and Lifecycle

## Status
Accepted (Updated 2026-01-17)

## Date
Original: 2026-01-07
Updated: 2026-01-09, 2026-01-11, 2026-01-16, 2026-01-17

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
| CTP | Chained Tickerplant (batched publisher) |
| HDB | Historical Database |
| MLE | Machine Learning Engine |
| RDB | Real-time Database (query-only) |
| RTE | Real-Time Engine |
| TEL | Telemetry Process |
| TP | Tickerplant |
| WDB | Write-only Database (intraday writedown to HDB) |

## Decision

An **on-demand log manager** (`logmgr.q`) handles log lifecycle management.

### Architecture

```
                    TP:5010
                       │
                       ▼
              logs/YYYY.MM.DD.log
             (trades + quotes)
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      WDB:5011     MLE:5012     CTP:5014
      (replay)     (replay)         │
                              ┌─────┼─────┐
                              ▼     ▼     ▼
                          RTE:5015 TEL:5016 RDB:5017
                          (live)  (live)  (live)

                  +── logmgr.q (on-demand) ──+
                  │  .log.list[]             │
                  │  .log.cleanup[days]      │
                  │  .log.verify[file]       │
                  +──────────────────────────+
```

### Log Manager Responsibilities

| Function | Description |
|----------|-------------|
| Discovery | Find and list all log files |
| Diagnostics | Report sizes, chunk counts, dates |
| Verification | Check log file integrity |
| Cleanup | Delete logs older than retention period |
| Monitoring | Provide log health metrics |

### On-Demand Usage

Log management is run manually when needed:

```bash
# Start logmgr interactively
cd ~/new-tick-to-signal/kdb
q logmgr.q

# Or run specific commands
q logmgr.q -q <<< ".log.list[]"
q logmgr.q -q <<< ".log.cleanup[7]"
```

**No port assignment:** logmgr does not listen on a port and is not a continuously running service.

### Query Interface

**Discovery:**
```q
.log.list[]
/ Returns table:
/ date       file                     sizeMB   chunks
/ --------------------------------------------------------
/ 2026.01.06 `:logs/2026.01.06.log    141.97   359740
/ 2026.01.07 `:logs/2026.01.07.log    70.21    366586

.log.summary[]
/ Returns summary by date
```

**Verification:**
```q
.log.verifyDate[.z.D]
/ Verifies today's log

.log.verify[`:logs/2026.01.07.log]
/ Verifies specific file
```

**Cleanup:**
```q
.log.cleanup[7]    / Delete logs older than 7 days
.log.cleanup[]     / Use default retention (7 days)
```

### Retention Policy

| Parameter | Default | Description |
|-----------|---------|-------------|
| `retentionDays` | 7 | Keep logs for 7 days |

## Rationale

This approach was selected because:

**On-demand tool:**
- Separates log management from data processing
- No additional running process to manage
- Minimal resource usage
- Can be scheduled via cron if needed

**Query-based interface:**
- Flexible (any retention period, any date)
- Scriptable (can be called from external tools)
- Transparent (shows what will be deleted)

## Consequences

### Positive

- Clear visibility into log file status
- Controlled disk space usage
- Easy diagnostics and verification
- Flexible retention policies
- No impact on data processing components
- No additional running process

### Negative / Trade-offs

- Manual cleanup required (no automatic by default)
- Must remember to run cleanup periodically

## Links / References

- `adr-003-tickerplant-logging-and-durability-strategy.md` (log file format, single file)
- `adr-006-recovery-and-replay-strategy.md` (replay mechanism)

# Application Log Analysis Report Template

Template for the markdown report generated in Step 7b. All `{PLACEHOLDERS}` are
replaced at generation time.

## File Naming

```bash
TIMESTAMP=$(TZ=Asia/Shanghai date +%Y-%m-%d-%H-%M-%S)
REPORT_FILE="${EXPERIMENT_DIR}/${TIMESTAMP}-app-log-analysis.md"
```

---

## Report Structure

```markdown
# Application Log Analysis Report

**Experiment ID:** {EXPERIMENT_ID}
**Analysis Time:** {TIMESTAMP}
**Time Range:** {START_TIME} - {END_TIME} (includes 3-min post-experiment baseline)
**Duration:** {DURATION}

## Summary

| Service | Application | Total Errors | Peak Error Rate | Recovery Time |
|---------|-------------|--------------|-----------------|---------------|
| {service} | {app} | {count} | {rate}/min | {time} |

## Per-Service Application Analysis

### {Service Name} ({resource_id})

#### {Application Name} ({namespace}/{deployment})

**Error Timeline:**

| Time (UTC) | Level | Message |
|------------|-------|---------|
| {HH:MM:SS} | ERROR | {truncated message} |
| ... | ... | ... |

**Key Error Patterns:**

| Pattern | Count | First Occurrence | Last Occurrence |
|---------|-------|------------------|-----------------|
| Connection refused | {n} | {time} | {time} |
| Timeout | {n} | {time} | {time} |

**Log Sample (Critical Errors):**

```
{5-10 lines of actual error logs}
```

**Insights:**
- {insight_1}: Error spike at {time}, correlates with {service} failover
- {insight_2}: Recovery detected at {time}, {duration} after fault injection ended
- {insight_3}: Application retry mechanism worked/failed because...

(Repeat for each application)

## Cross-Service Correlation

| Time | Event | RDS Impact | ElastiCache Impact | Application Response |
|------|-------|------------|--------------------|--------------------|
| {time} | Fault injection start | - | - | First errors appear |
| {time} | {service} failover | Connection errors | - | Retrying... |
| {time} | Recovery | Connections restored | - | Normal operation |
| {time} | Experiment ended | - | - | - |
| --- | **Post-baseline window (3 min)** | --- | --- | --- |
| {time} | Baseline collection start | Normal | Normal | Steady state |
| {time} | Baseline collection end | Normal | Normal | Steady state |

## Managed Service Log Insights

(Include this section ONLY if Step 3.5 collected managed service logs.)

### {Service Name} ({resource_id})

**Logging status:** Enabled ({log-types})
**Log group:** {log-group-name}

**Key Events:**

| Time (UTC) | Event |
|------------|-------|
| {HH:MM:SS} | {event description, e.g., "Failover started", "Node marked NotReady"} |

**Correlation with Application Logs:**
- {insight}: {service} failover at {time} correlates with application connection errors at {time}
- {insight}: Application recovery at {time} is {N} seconds after {service} recovery at {time}

(If logging was NOT enabled for a service, list it here with a recommendation to enable.)

## Recommendations

1. **{Issue}:** {description}
   - **Impact:** {what happened}
   - **Recommendation:** {what to improve}

## Appendix: Log File Locations

**Raw log directory:** `{LOG_DIR}`

| Application | Log File |
|-------------|----------|
| {app} | `{LOG_DIR}/{service}/{app}.log` |
```

---
title: LogQL Query Examples
date: "2025-11-24"
weight: 4
---

# LogQL Query Examples for SonataFlow Workflows

This page provides comprehensive LogQL query examples for analyzing SonataFlow workflow logs in Loki/Grafana.

## LogQL Basics

LogQL is the query language for Loki, similar to PromQL for Prometheus. It consists of:

1. **Log Stream Selector**: `{label="value"}` - Selects log streams
2. **Log Pipeline**: `| operation` - Processes log lines
3. **Aggregations**: `sum()`, `count()`, `rate()` - Aggregate results

### Basic Syntax

```
{<label selectors>} |= "<search string>" | <log pipeline> | <aggregation>
```

## Common Use Cases

### 1. Filtering Logs by Process Instance

Get all logs for a specific process instance:

```logql
{job="sonataflow-workflows", processInstanceId="abc-123-456"}
```

With JSON parsing:

```logql
{job="sonataflow-workflows", processInstanceId="abc-123-456"}
| json
| line_format "{{.timestamp}} [{{.level}}] {{.state}}: {{.message}}"
```

Get logs for multiple process instances:

```logql
{job="sonataflow-workflows", processInstanceId=~"abc-123.*|def-456.*"}
```

### 2. Finding Errors for a Workflow

All errors for a specific workflow:

```logql
{job="sonataflow-workflows", workflow_name="onboarding", level="ERROR"}
```

Errors with exception details:

```logql
{job="sonataflow-workflows", workflow_name="onboarding", level="ERROR"}
| json
| exception != ""
| line_format "Error: {{.message}}\nException: {{.exception}}"
```

Count errors by workflow:

```logql
sum by (workflow_name) (count_over_time({job="sonataflow-workflows", level="ERROR"}[1h]))
```

Error rate per minute:

```logql
sum by (workflow_name) (rate({job="sonataflow-workflows", level="ERROR"}[5m]))
```

### 3. Correlating Logs with Traces

Find logs by trace ID:

```logql
{job="sonataflow-workflows", traceId="1a2b3c4d5e6f7g8h"}
```

Get all logs for a trace with span information:

```logql
{job="sonataflow-workflows", traceId="1a2b3c4d5e6f7g8h"}
| json
| line_format "{{.timestamp}} [Span: {{.spanId}}] {{.state}}: {{.message}}"
```

Find traces with errors:

```logql
{job="sonataflow-workflows", level="ERROR"}
| json
| traceId != ""
| distinct traceId
```

### 4. Monitoring Workflow Duration

Extract duration from logs (if logged):

```logql
{job="sonataflow-workflows", message=~".*completed.*"}
| json
| duration > 10
```

Calculate average workflow execution time:

```logql
avg_over_time({job="sonataflow-workflows"}
| json
| unwrap duration [1h]) by (workflow_name)
```

Find slow workflows (p95):

```logql
quantile_over_time(0.95, {job="sonataflow-workflows"}
| json
| unwrap duration [1h]) by (workflow_name)
```

Histogram of workflow durations:

```logql
histogram_over_time({job="sonataflow-workflows", message=~".*completed.*"}
| json
| unwrap duration [1h])
```

## Advanced Queries

### 5. Process Instance Lifecycle

Track process instance from start to completion:

```logql
{job="sonataflow-workflows", processInstanceId="abc-123"}
| json
| line_format "{{.timestamp}} | {{.state}} | {{.message}}"
```

Count process instances by state:

```logql
sum by (state) (count_over_time({job="sonataflow-workflows", workflow_name="onboarding"}
| json
| state != "" [1h]))
```

Find failed process instances:

```logql
{job="sonataflow-workflows", message=~".*failed.*|.*error.*"}
| json
| processInstanceId != ""
| distinct processInstanceId
```

### 6. Workflow State Analysis

Get state transition frequency:

```logql
sum by (state) (count_over_time({job="sonataflow-workflows", workflow_name="onboarding"}
| json
[1h]))
```

Find workflows stuck in a state:

```logql
{job="sonataflow-workflows", state="ApprovalStep"}
| json
| __timestamp__ < (now() - 1h)
```

State duration (time spent in each state):

```logql
{job="sonataflow-workflows", processInstanceId="abc-123"}
| json
| line_format "{{.state}}: {{.timestamp}}"
```

### 7. Error Pattern Detection

Find common error messages:

```logql
topk(10, sum by (message) (count_over_time({job="sonataflow-workflows", level="ERROR"}
| json
[24h])))
```

Errors by exception type:

```logql
sum by (exception) (count_over_time({job="sonataflow-workflows", level="ERROR"}
| json
| exception != "" [1h]))
```

Find cascading failures:

```logql
{job="sonataflow-workflows", level="ERROR"}
| json
| traceId != ""
| count by (traceId) > 3
```

### 8. Performance Metrics

Throughput (workflows per minute):

```logql
sum(rate({job="sonataflow-workflows", message=~".*started.*"}[5m])) by (workflow_name)
```

Success rate:

```logql
(
  sum(rate({job="sonataflow-workflows", message=~".*completed.*"}[5m])) by (workflow_name)
  /
  sum(rate({job="sonataflow-workflows", message=~".*started.*"}[5m])) by (workflow_name)
) * 100
```

Error rate percentage:

```logql
(
  sum(rate({job="sonataflow-workflows", level="ERROR"}[5m])) by (workflow_name)
  /
  sum(rate({job="sonataflow-workflows"}[5m])) by (workflow_name)
) * 100
```

### 9. Resource Usage Correlation

Logs by pod:

```logql
sum by (pod, namespace) (count_over_time({job="sonataflow-workflows"}[5m]))
```

Find pods with high error rates:

```logql
topk(5, sum by (pod) (rate({job="sonataflow-workflows", level="ERROR"}[5m])))
```

Compare log volume across namespaces:

```logql
sum by (namespace) (count_over_time({job="sonataflow-workflows"}[1h]))
```

### 10. User Activity Tracking

Logs by user (if userId is in MDC):

```logql
{job="sonataflow-workflows"}
| json
| userId != ""
| line_format "User {{.userId}}: {{.message}}"
```

Top active users:

```logql
topk(10, sum by (userId) (count_over_time({job="sonataflow-workflows"}
| json
| userId != "" [24h])))
```

User journey tracking:

```logql
{job="sonataflow-workflows", processInstanceId="abc-123"}
| json
| line_format "{{.timestamp}} | User: {{.userId}} | {{.state}} | {{.message}}"
```

## Log Pipeline Operations

### JSON Parsing

Parse JSON logs:

```logql
{job="sonataflow-workflows"} | json
```

Parse specific fields:

```logql
{job="sonataflow-workflows"}
| json message, level, processInstanceId, traceId
```

Parse nested fields:

```logql
{job="sonataflow-workflows"}
| json
| json processInstanceId=MDC.processInstanceId
```

### Line Formatting

Format output:

```logql
{job="sonataflow-workflows"}
| json
| line_format "{{.timestamp}} [{{.level}}] {{.logger}}: {{.message}}"
```

Conditional formatting:

```logql
{job="sonataflow-workflows"}
| json
| line_format "{{ if eq .level \"ERROR\" }}🔴{{ else }}✅{{ end }} {{.message}}"
```

### Pattern Matching

Regex filter (include):

```logql
{job="sonataflow-workflows"} |~ "workflow.*started"
```

Regex filter (exclude):

```logql
{job="sonataflow-workflows"} !~ "health.*check"
```

Multiple patterns:

```logql
{job="sonataflow-workflows"} |~ "error|failed|exception"
```

Case-insensitive search:

```logql
{job="sonataflow-workflows"} |~ "(?i)error"
```

### Label Filters

Filter by label values:

```logql
{job="sonataflow-workflows", workflow_name=~"onboarding|offboarding"}
```

Exclude label values:

```logql
{job="sonataflow-workflows", level!="DEBUG"}
```

Multiple label conditions:

```logql
{job="sonataflow-workflows", workflow_name="onboarding", namespace="production", level="ERROR"}
```

### Unwrap and Aggregations

Extract numeric values:

```logql
{job="sonataflow-workflows"}
| json
| unwrap duration
```

Sum unwrapped values:

```logql
sum_over_time({job="sonataflow-workflows"}
| json
| unwrap duration [1h]) by (workflow_name)
```

Average:

```logql
avg_over_time({job="sonataflow-workflows"}
| json
| unwrap duration [1h])
```

Min/Max:

```logql
max_over_time({job="sonataflow-workflows"}
| json
| unwrap duration [1h]) by (workflow_name)
```

## Metrics from Logs

### Count Queries

Count log lines:

```logql
count_over_time({job="sonataflow-workflows"}[5m])
```

Count by label:

```logql
sum by (workflow_name) (count_over_time({job="sonataflow-workflows"}[1h]))
```

Count distinct values:

```logql
count(count by (processInstanceId) (count_over_time({job="sonataflow-workflows"}[1h])))
```

### Rate Queries

Logs per second:

```logql
rate({job="sonataflow-workflows"}[5m])
```

Errors per second by workflow:

```logql
sum by (workflow_name) (rate({job="sonataflow-workflows", level="ERROR"}[5m]))
```

### Bytes Queries

Log volume in bytes:

```logql
bytes_rate({job="sonataflow-workflows"}[5m])
```

Total bytes over time:

```logql
bytes_over_time({job="sonataflow-workflows"}[1h])
```

## Real-World Query Examples

### Example 1: SLA Monitoring

Find workflows exceeding SLA (> 5 minutes):

```logql
{job="sonataflow-workflows", message=~".*completed.*"}
| json
| duration > 300
| line_format "⚠️ SLA breach: {{.processInstanceId}} took {{.duration}}s"
```

SLA compliance rate:

```logql
(
  count_over_time({job="sonataflow-workflows", message=~".*completed.*"}
  | json
  | duration <= 300 [24h])
  /
  count_over_time({job="sonataflow-workflows", message=~".*completed.*"} [24h])
) * 100
```

### Example 2: Debugging Failed Workflows

Get full context for failed workflow:

```logql
{job="sonataflow-workflows", processInstanceId="abc-123"}
| json
| line_format "{{.timestamp}} | {{.level}} | {{.state}} | {{.message}} | {{.exception}}"
```

Find all failures in last hour:

```logql
{job="sonataflow-workflows", level="ERROR", message=~".*failed.*"}
| json
| line_format "Workflow: {{.workflowId}}, Instance: {{.processInstanceId}}, Error: {{.message}}"
```

### Example 3: Capacity Planning

Peak concurrent workflows:

```logql
max_over_time(
  count(count by (processInstanceId) ({job="sonataflow-workflows"})) [24h:1m]
)
```

Workflow distribution by hour:

```logql
sum by (hour) (count_over_time({job="sonataflow-workflows", message=~".*started.*"}
| json
| __timestamp__ [24h]))
```

### Example 4: Audit Trail

Complete audit log for process instance:

```logql
{job="sonataflow-workflows", processInstanceId="abc-123"}
| json
| line_format "{{.timestamp}} | User: {{.userId}} | State: {{.state}} | Action: {{.message}}"
```

User actions across workflows:

```logql
{job="sonataflow-workflows"}
| json
| userId="user@example.com"
| line_format "{{.timestamp}} | Workflow: {{.workflowId}} | Instance: {{.processInstanceId}} | {{.message}}"
```

### Example 5: Integration Monitoring

External API call errors:

```logql
{job="sonataflow-workflows", message=~".*REST.*error.*|.*HTTP.*[45][0-9]{2}.*"}
| json
| line_format "API Error: {{.message}}"
```

Integration latency:

```logql
avg_over_time({job="sonataflow-workflows", message=~".*REST call completed.*"}
| json
| unwrap duration [1h]) by (endpoint)
```

### Example 6: Data Quality Checks

Find validation errors:

```logql
{job="sonataflow-workflows", message=~".*validation.*failed.*"}
| json
| line_format "Validation Error in {{.state}}: {{.message}}"
```

Missing required fields:

```logql
{job="sonataflow-workflows", exception=~".*NullPointerException.*|.*required.*field.*"}
| json
```

## Alerting Queries

### Alert on High Error Rate

```logql
sum by (workflow_name) (rate({job="sonataflow-workflows", level="ERROR"}[5m])) > 0.1
```

Alert definition:
- **Condition**: Error rate > 0.1 errors/sec for 5 minutes
- **Severity**: Warning
- **Action**: Notify on-call team

### Alert on Workflow Failures

```logql
count_over_time({job="sonataflow-workflows", message=~".*workflow.*failed.*"}[5m]) > 0
```

Alert definition:
- **Condition**: Any workflow failure
- **Severity**: Critical
- **Action**: Page on-call engineer

### Alert on Long-Running Workflows

```logql
count({job="sonataflow-workflows", state="InProgress"}
| json
| __timestamp__ < (now() - 3600)) by (processInstanceId) > 0
```

Alert definition:
- **Condition**: Workflow in same state for > 1 hour
- **Severity**: Warning
- **Action**: Create ticket for investigation

### Alert on Missing Process Definitions

```logql
absent_over_time({job="sonataflow-workflows", workflow_name="critical-workflow"}[10m])
```

Alert definition:
- **Condition**: No logs from critical workflow for 10 minutes
- **Severity**: Critical
- **Action**: Check if workflow pod is running

## Query Optimization Tips

### 1. Use Label Filters First

**Good**:
```logql
{job="sonataflow-workflows", workflow_name="onboarding", level="ERROR"}
```

**Bad** (slower):
```logql
{job="sonataflow-workflows"} | json | workflow_name="onboarding" | level="ERROR"
```

### 2. Limit Time Range

**Good**:
```logql
{job="sonataflow-workflows"}[1h]
```

**Bad**:
```logql
{job="sonataflow-workflows"}[30d]  # Very slow
```

### 3. Use Specific Matchers

**Good**:
```logql
{job="sonataflow-workflows", processInstanceId="abc-123"}
```

**Bad**:
```logql
{job="sonataflow-workflows"} |~ "abc-123"
```

### 4. Aggregate Early

**Good**:
```logql
sum by (workflow_name) (count_over_time({job="sonataflow-workflows"}[5m]))
```

**Bad**:
```logql
{job="sonataflow-workflows"} | json | count by (workflow_name)
```

### 5. Use Appropriate Step Size

For dashboards, use `$__interval`:
```logql
count_over_time({job="sonataflow-workflows"}[$__interval])
```

## Testing Queries

### Using Grafana Explore

1. Navigate to **Explore** in Grafana
2. Select **Loki** datasource
3. Enter your LogQL query
4. Click **Run Query**
5. View results and adjust query

### Using Loki API

```bash
# Query API
curl -G -s "http://loki:3100/loki/api/v1/query" \
  --data-urlencode 'query={job="sonataflow-workflows"}' \
  --data-urlencode 'limit=10' | jq

# Query range API
curl -G -s "http://loki:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="sonataflow-workflows"}' \
  --data-urlencode 'start=2025-11-24T00:00:00Z' \
  --data-urlencode 'end=2025-11-24T23:59:59Z' \
  --data-urlencode 'limit=100' | jq
```

### Using logcli

```bash
# Install logcli
curl -O -L "https://github.com/grafana/loki/releases/download/v2.9.3/logcli-linux-amd64.zip"
unzip logcli-linux-amd64.zip

# Query logs
./logcli query '{job="sonataflow-workflows"}' \
  --addr=http://loki:3100 \
  --limit=50

# Query with time range
./logcli query '{job="sonataflow-workflows", level="ERROR"}' \
  --addr=http://loki:3100 \
  --from="2025-11-24T00:00:00Z" \
  --to="2025-11-24T23:59:59Z"

# Follow logs (tail)
./logcli query '{job="sonataflow-workflows"}' \
  --addr=http://loki:3100 \
  --tail \
  --follow
```

## Common Errors and Solutions

### Error: "parse error: unexpected <character>"

**Cause**: Invalid LogQL syntax

**Solution**: Check for:
- Unmatched braces `{}` or brackets `[]`
- Missing pipes `|`
- Invalid regex patterns

### Error: "too many outstanding requests"

**Cause**: Query too broad or cluster overloaded

**Solution**:
- Narrow time range
- Add more label filters
- Reduce query parallelism

### Error: "maximum of series (X) reached"

**Cause**: Too many unique label combinations

**Solution**:
- Use fewer labels in `by()` clause
- Increase Loki limits
- Aggregate earlier in pipeline

## Reference

### LogQL Operators

- `|=` : Contains string
- `!=` : Does not contain string
- `|~` : Regex match
- `!~` : Regex does not match
- `>`, `>=`, `<`, `<=`, `==`, `!=` : Numeric comparisons

### Aggregation Functions

- `sum()` : Sum values
- `avg()` : Average values
- `min()` : Minimum value
- `max()` : Maximum value
- `count()` : Count entries
- `stddev()` : Standard deviation
- `stdvar()` : Standard variance
- `topk()` : Top K values
- `bottomk()` : Bottom K values

### Range Vector Functions

- `rate()` : Per-second rate
- `count_over_time()` : Count in range
- `bytes_rate()` : Bytes per second
- `bytes_over_time()` : Total bytes
- `sum_over_time()` : Sum in range
- `avg_over_time()` : Average in range
- `max_over_time()` : Maximum in range
- `min_over_time()` : Minimum in range
- `quantile_over_time()` : Quantile in range

## Next Steps

- [View Dashboard](grafana-dashboard.md) to visualize queries
- [Configure Alerts](grafana-dashboard.md#alerting-configuration) based on queries
- [Deploy RBAC](deployment-manifests.md) for secure access
- [Optimize Promtail](promtail-config.md) for better performance

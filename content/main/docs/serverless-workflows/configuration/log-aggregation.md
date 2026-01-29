---
title: "Configure Log Aggregation for Serverless Workflows"
date: 2025-11-24
---

Configure structured JSON logging and log aggregation for SonataFlow workflows to enable comprehensive monitoring, debugging, and observability across your workflow instances.

# Prerequisites

* OpenShift Container Platform 4.8+ or Kubernetes 1.21+
* SonataFlow workflows deployed via SonataFlow Operator
* Cluster admin permissions for deploying log aggregation stack
* Basic knowledge of JSON logging and log aggregation tools

# Overview

SonataFlow workflows support structured JSON logging with automatic process instance correlation through:

• **Process Instance Context**: Automatic `processInstanceId` correlation in all log entries
• **Structured Format**: JSON logs optimized for machine processing and aggregation
• **Multi-tenancy Support**: Log isolation by workflow and process instance

## JSON Log Structure

When properly configured, workflow logs are emitted as JSON with the following structure:

```json
{
  "timestamp": "2025-11-24T10:30:45.123Z",
  "level": "INFO",
  "loggerName": "org.kie.kogito.workflow.engine",
  "message": "Workflow step completed successfully",
  "threadName": "executor-thread-1",
  "mdc": {
    "processInstanceId": "abc-123-def-456"
  }
}
```

# Configuration

## Workflow Configuration

To enable JSON logging for your workflows, configure the following properties in the `{workflow-name}-props` ConfigMap:

```properties
# Enable JSON logging with Quarkus JSON logging extension
quarkus.log.console.json=true
quarkus.log.console.json.pretty-print=false

# Include all MDC context fields in JSON output
# - processInstanceId: Set automatically by SonataFlow/Kogito
# - traceId, spanId: Set by Quarkus OpenTelemetry (requires quarkus.otel.enabled=true)
quarkus.log.console.json.print-details=true

# Configure log levels for workflow components
quarkus.log.category."org.kie.kogito".level=DEBUG
quarkus.log.category."io.serverlessworkflow".level=INFO
quarkus.log.category."org.kie.kogito.process".level=INFO

# Optional: Enable additional context logging
quarkus.log.category."org.kie.kogito.services.context".level=DEBUG
```

## Required Extension

Add the Quarkus JSON logging extension in the `QUARKUS_EXTENSIONS` environment variable when building the workflow image:

```
export QUARKUS_EXTENSIONS="${QUARKUS_EXTENSIONS},io.quarkus:quarkus-logging-json"
```

**Note**: This extension is **required** for JSON log formatting. Without it, logs will remain in plain text format.

## File-Based JSON Logging

In some deployment scenarios, you may need to output logs to a file instead of (or in addition to) the console. This is useful when:

- Using sidecar containers that read log files (e.g., Fluent Bit file input)
- Integrating with legacy log collection systems that expect file-based logs
- Debugging locally with persistent log files
- Collecting logs from environments where stdout/stderr collection is limited

### Basic File Logging Configuration

Add the following properties to enable JSON logging to a file:

```properties
# Enable file logging
quarkus.log.file.enable=true
quarkus.log.file.path=/var/log/sonataflow/workflow.log

# Enable JSON format for file output
quarkus.log.file.json=true
quarkus.log.file.json.pretty-print=false

# Include MDC context fields in JSON output
# - processInstanceId: Set automatically by SonataFlow/Kogito
# - traceId, spanId: Set by Quarkus OpenTelemetry (requires quarkus.otel.enabled=true)
quarkus.log.file.json.print-details=true

# Set log level for file output
quarkus.log.file.level=INFO
```

### File Rotation Configuration

For production environments, configure log rotation to prevent disk space issues:

```properties
# Enable file logging with rotation
quarkus.log.file.enable=true
quarkus.log.file.path=/var/log/sonataflow/workflow.log

# JSON format
quarkus.log.file.json=true
quarkus.log.file.json.pretty-print=false
quarkus.log.file.json.print-details=true

# Rotation settings
quarkus.log.file.rotation.max-file-size=10M
quarkus.log.file.rotation.max-backup-index=5
quarkus.log.file.rotation.file-suffix=.yyyy-MM-dd
quarkus.log.file.rotation.rotate-on-boot=true
```

This configuration:
- Rotates logs when they reach 10MB
- Keeps up to 5 backup files
- Adds date suffix to rotated files
- Rotates on application startup

### Combined Console and File Logging

You can enable both console and file JSON logging simultaneously:

```properties
# Console JSON logging (for container log collectors)
quarkus.log.console.json=true
quarkus.log.console.json.pretty-print=false
quarkus.log.console.json.print-details=true

# File JSON logging (for file-based collectors)
quarkus.log.file.enable=true
quarkus.log.file.path=/var/log/sonataflow/workflow.log
quarkus.log.file.json=true
quarkus.log.file.json.pretty-print=false
quarkus.log.file.json.print-details=true
quarkus.log.file.rotation.max-file-size=10M
quarkus.log.file.rotation.max-backup-index=5
```

### Kubernetes Volume Configuration

When using file-based logging in Kubernetes, ensure the log directory is properly mounted:

```yaml
apiVersion: sonataflow.org/v1alpha08
kind: SonataFlow
metadata:
  name: my-workflow
spec:
  podTemplate:
    container:
      volumeMounts:
      - name: logs
        mountPath: /var/log/sonataflow
    volumes:
    - name: logs
      emptyDir: {}
```

For persistent logs or sidecar collection, use a shared volume:

```yaml
spec:
  podTemplate:
    container:
      volumeMounts:
      - name: shared-logs
        mountPath: /var/log/sonataflow
    initContainers: []
    volumes:
    - name: shared-logs
      emptyDir:
        sizeLimit: 500Mi
```

## Validation

Before deploying log aggregation, verify that JSON logging is working correctly:

### 1. Check Workflow Pod Logs

After applying the configuration, restart your workflow pod and check the log output:

```bash
# Get workflow pod name
oc get pods -n sonataflow-infra -l sonataflow.org/workflow-app=your-workflow

# Check logs for JSON format
oc logs -n sonataflow-infra your-workflow-pod-name | head -5
```

**Expected output** (JSON format):
```json
{"timestamp":"2025-11-24T10:30:45.123Z","level":"INFO","logger":"io.quarkus","message":"Profile prod activated","MDC":{}}
```

**If you see plain text** instead:
```
2025-11-24 10:30:45,123 INFO  [io.quarkus] Profile prod activated
```

This indicates the `quarkus-logging-json` extension is not properly included in your workflow image.

### 2. Verify MDC Context

Look for workflow-specific logs that include `processInstanceId`:

```bash
# Search for logs with process instance context
oc logs your-workflow-pod-name | grep processInstanceId
```

**Expected**: JSON logs containing `"MDC":{"processInstanceId":"abc-123-..."}`

**If MDC fields are empty or missing**: The JSON logging is working, but process context is not being set. This may indicate:
- Workflow has not processed any instances yet
- Custom MDC configuration may be needed depending on your SonataFlow version

### 3. Test with a Simple Workflow Execution

Trigger a workflow execution and verify the logs contain process correlation:

```bash
# Trigger workflow (method depends on your setup)
curl -X POST http://your-workflow-url/your-workflow

# Check logs immediately after
oc logs your-workflow-pod-name --tail=20 | grep processInstanceId
```

## ConfigMap Example

Here's a complete example of a workflow ConfigMap with JSON logging enabled:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: greetings-props
  namespace: sonataflow-infra
data:
  application.properties: |
    # JSON logging configuration
    quarkus.log.console.json=true
    quarkus.log.console.json.pretty-print=false
    quarkus.log.console.json.print-details=true

    # Log levels
    quarkus.log.category."org.kie.kogito".level=DEBUG
    quarkus.log.category."io.serverlessworkflow".level=INFO
```

# Log Aggregation Solutions

## Recommended Stack: Promtail + Loki + Grafana (PLG)

The PLG stack provides a modern, cloud-native solution optimized for Kubernetes environments.

> 💡 **For detailed production deployment guides**: See the comprehensive [PLG Observability Documentation](../observability/) which includes complete Helm values, YAML manifests, dashboard configurations, and advanced query examples.

### Architecture

```
SonataFlow Pods → Promtail (log collection) → Loki (storage) → Grafana (visualization)
```

### Quick Deployment

For a quick start, deploy the PLG stack using Helm:

```bash
# Add Grafana Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create namespace
oc new-project sonataflow-observability

# Deploy Loki stack
helm install loki-stack grafana/loki-stack \
  --namespace sonataflow-observability \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=20Gi \
  --set promtail.config.logLevel=info \
  --set grafana.enabled=true
```

> 📋 **For production deployment**: Use the [complete Helm values configuration](../observability/helm-values/) with proper resource limits, security contexts, and OpenShift-specific settings.

### Promtail Configuration

Configure Promtail to discover and parse SonataFlow logs. You can choose between scraping container stdout (default) or custom JSON log files.

#### Option A: Scrape Container Stdout (Default)

This configuration uses Kubernetes service discovery to collect logs from container stdout:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: sonataflow-observability
data:
  config.yml: |
    server:
      http_listen_port: 3101

    clients:
      - url: http://loki:3100/loki/api/v1/push

    scrape_configs:
    - job_name: sonataflow-workflows
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ["sonataflow-infra"]

      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_sonataflow_org_workflow_app]
        action: keep
        regex: (.+)

      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

      - source_labels: [__meta_kubernetes_pod_label_sonataflow_org_workflow_app]
        target_label: workflow

      pipeline_stages:
      - json:
          expressions:
            timestamp: timestamp
            level: level
            logger: logger
            message: message
            processInstanceId: mdc.processInstanceId
            traceId: mdc.traceId
            spanId: mdc.spanId

      - labels:
          level:
          logger:
          processInstanceId:
          traceId:
```

#### Option B: Scrape JSON Log Files

When using [file-based JSON logging](#file-based-json-logging), configure Promtail as a sidecar to read from the shared log volume:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-sidecar-config
  namespace: sonataflow-infra
data:
  config.yml: |
    server:
      http_listen_port: 3101

    clients:
      - url: http://loki.sonataflow-observability.svc.cluster.local:3100/loki/api/v1/push

    positions:
      filename: /var/log/positions.yaml

    scrape_configs:
    - job_name: sonataflow-json-files
      static_configs:
      - targets:
          - localhost
        labels:
          job: sonataflow-workflows
          __path__: /var/log/sonataflow/*.log

      pipeline_stages:
      - json:
          expressions:
            timestamp: timestamp
            level: level
            logger: loggerName
            message: message
            processInstanceId: mdc.processInstanceId
            traceId: mdc.traceId
            spanId: mdc.spanId

      - labels:
          level:
          logger:
          processInstanceId:
          traceId:

      - timestamp:
          source: timestamp
          format: RFC3339Nano
```

**Promtail Sidecar Deployment:**

Add Promtail as a sidecar container in your SonataFlow CR:

```yaml
apiVersion: sonataflow.org/v1alpha08
kind: SonataFlow
metadata:
  name: my-workflow
  namespace: sonataflow-infra
spec:
  podTemplate:
    container:
      volumeMounts:
      - name: shared-logs
        mountPath: /var/log/sonataflow
    containers:
    - name: promtail-sidecar
      image: grafana/promtail:2.9.0
      args:
        - -config.file=/etc/promtail/config.yml
      volumeMounts:
      - name: shared-logs
        mountPath: /var/log/sonataflow
        readOnly: true
      - name: promtail-config
        mountPath: /etc/promtail
      - name: positions
        mountPath: /var/log
      resources:
        requests:
          cpu: 50m
          memory: 64Mi
        limits:
          cpu: 100m
          memory: 128Mi
    volumes:
    - name: shared-logs
      emptyDir:
        sizeLimit: 500Mi
    - name: promtail-config
      configMap:
        name: promtail-sidecar-config
    - name: positions
      emptyDir: {}
```

### Query Examples

**Filter logs by process instance:**
```logql
{job="sonataflow-workflows"} | json | processInstanceId="abc-123-def-456"
```

**Find workflow errors:**
```logql
{job="sonataflow-workflows", workflow="onboarding"} | json | level="ERROR"
```

**Trace correlation:**
```logql
{job="sonataflow-workflows"} | json | traceId="4bf92f3577b34da6a3ce929d0e0e4736"
```

**Process instance timeline:**
```logql
{job="sonataflow-workflows"} | json | processInstanceId="abc-123-def-456" | line_format "{{.timestamp}} [{{.level}}] {{.message}}"
```

> 🔍 **50+ additional query examples**: See [LogQL Query Reference](../observability/logql-queries/) for comprehensive examples including SLA monitoring, capacity planning, and audit trails.

## Alternative Stack: Fluent Bit + OpenSearch

For organizations preferring Elasticsearch-compatible solutions:

### Deployment

```bash
# Deploy OpenSearch cluster
helm repo add opensearch https://opensearch-project.github.io/helm-charts/
helm install opensearch opensearch/opensearch \
  --namespace logging \
  --set clusterName=sonataflow-logs \
  --set nodeGroup=master \
  --set masterService=opensearch \
  --set replicas=3

# Deploy Fluent Bit
helm install fluent-bit fluent/fluent-bit \
  --namespace logging \
  --set outputs.es.host=opensearch \
  --set outputs.es.index=sonataflow-logs
```

### Fluent Bit Configuration

You can configure Fluent Bit to collect logs from container stdout or from custom JSON log files.

#### Option A: Scrape Container Logs (Default)

This configuration collects logs from the standard Kubernetes container log location:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  custom_parsers.conf: |
    [PARSER]
        Name        sonataflow_json
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z

  fluent-bit.conf: |
    [INPUT]
        Name              tail
        Path              /var/log/containers/*sonataflow*.log
        Parser            sonataflow_json
        Tag               kube.sonataflow.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB

    [FILTER]
        Name                kubernetes
        Match               kube.sonataflow.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Keep_Log            On
        Merge_Log           On

    [OUTPUT]
        Name            es
        Match           kube.sonataflow.*
        Host            opensearch
        Port            9200
        Index           sonataflow-logs-%Y.%m.%d
        Type            _doc
        Logstash_Format On
```

#### Option B: Scrape JSON Log Files (Sidecar)

When using [file-based JSON logging](#file-based-json-logging), deploy Fluent Bit as a sidecar to read from the shared log volume:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-sidecar-config
  namespace: sonataflow-infra
data:
  custom_parsers.conf: |
    [PARSER]
        Name        sonataflow_json
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z

  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Parsers_File  /fluent-bit/etc/custom_parsers.conf

    [INPUT]
        Name              tail
        Path              /var/log/sonataflow/*.log
        Parser            sonataflow_json
        Tag               sonataflow.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Read_from_Head    True
        DB                /var/log/flb_sonataflow.db

    [FILTER]
        Name          modify
        Match         sonataflow.*
        Add           kubernetes.namespace_name ${NAMESPACE}
        Add           kubernetes.pod_name ${POD_NAME}

    [OUTPUT]
        Name            es
        Match           sonataflow.*
        Host            opensearch.logging.svc.cluster.local
        Port            9200
        Index           sonataflow-logs-%Y.%m.%d
        Type            _doc
        Logstash_Format On
        Retry_Limit     5
```

**Fluent Bit Sidecar Deployment:**

Add Fluent Bit as a sidecar container in your SonataFlow CR:

```yaml
apiVersion: sonataflow.org/v1alpha08
kind: SonataFlow
metadata:
  name: my-workflow
  namespace: sonataflow-infra
spec:
  podTemplate:
    container:
      volumeMounts:
      - name: shared-logs
        mountPath: /var/log/sonataflow
    containers:
    - name: fluent-bit-sidecar
      image: fluent/fluent-bit:2.2
      env:
      - name: NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      volumeMounts:
      - name: shared-logs
        mountPath: /var/log/sonataflow
        readOnly: true
      - name: fluent-bit-config
        mountPath: /fluent-bit/etc
      - name: fluent-bit-db
        mountPath: /var/log
      resources:
        requests:
          cpu: 50m
          memory: 64Mi
        limits:
          cpu: 100m
          memory: 128Mi
    volumes:
    - name: shared-logs
      emptyDir:
        sizeLimit: 500Mi
    - name: fluent-bit-config
      configMap:
        name: fluent-bit-sidecar-config
    - name: fluent-bit-db
      emptyDir: {}
```

### OpenSearch Query Examples

**Process instance logs:**
```json
{
  "query": {
    "term": {
      "MDC.processInstanceId": "abc-123-def-456"
    }
  }
}
```

**Error aggregation:**
```json
{
  "query": {
    "bool": {
      "must": [
        {"term": {"level": "ERROR"}},
        {"range": {"@timestamp": {"gte": "now-1h"}}}
      ]
    }
  },
  "aggs": {
    "by_workflow": {
      "terms": {"field": "kubernetes.labels.sonataflow_org/workflow-app"}
    }
  }
}
```

# What's More: Visualization and Alerting

## Grafana Dashboards

Create comprehensive monitoring dashboards with:

• **Workflow Overview**: Active process instances, completion rates, error percentages
• **Process Timeline**: Step-by-step execution visualization per process instance
• **Performance Metrics**: Average duration, throughput, resource consumption
• **Error Analysis**: Error distribution, stack traces, failure patterns

> 📊 **Ready-to-use Dashboard**: Get a complete Grafana dashboard with 10 pre-built panels from [Grafana Dashboard Configuration](../observability/grafana-dashboard/).

### Sample Dashboard Panels

**Active Process Instances:**
```logql
count(count by (processInstanceId) ({job="sonataflow-workflows"} | json | processInstanceId != ""))
```

**Error Rate by Workflow:**
```logql
rate({job="sonataflow-workflows", level="ERROR"} | json[5m])
```

**Process Duration Distribution:**
```logql
histogram_quantile(0.95,
  rate({job="sonataflow-workflows"} | json | message="Workflow completed" | duration > 0[5m])
)
```

## Alerting Rules

Configure alerts for critical workflow conditions:

### Workflow Failures
```yaml
groups:
- name: sonataflow.rules
  rules:
  - alert: WorkflowHighErrorRate
    expr: rate({job="sonataflow-workflows", level="ERROR"}[5m]) > 0.1
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High error rate in SonataFlow workflows"
      description: "Error rate is {{ $value }} errors per second"

  - alert: WorkflowInstanceStuck
    expr: |
      time() - max by (process_instance_id) (
        {job="sonataflow-workflows"} | json | unwrap timestamp[1h]
      ) > 3600
    labels:
      severity: critical
    annotations:
      summary: "Workflow instance {{ $labels.process_instance_id }} appears stuck"
```

### Long-Running Processes
```yaml
  - alert: LongRunningWorkflow
    expr: |
      time() - min by (process_instance_id) (
        {job="sonataflow-workflows"} | json | message="Workflow started" | unwrap timestamp[24h]
      ) > 7200
    labels:
      severity: warning
    annotations:
      summary: "Workflow {{ $labels.process_instance_id }} running longer than 2 hours"
```

## Integration with External Systems

### OpenTelemetry Tracing
Correlate logs with distributed traces:

```properties
# Enable OpenTelemetry integration
quarkus.otel.exporter.otlp.traces.endpoint=http://jaeger-collector:14268/api/traces
quarkus.otel.service.name=${workflow.name}
quarkus.otel.resource.attributes=service.namespace=sonataflow-infra
```

### Notification Systems
Integrate with Slack, PagerDuty, or email:

```yaml
route:
  group_by: ['alertname', 'workflow']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'

receivers:
- name: 'web.hook'
  slack_configs:
  - api_url: 'YOUR_SLACK_WEBHOOK_URL'
    channel: '#workflow-alerts'
    title: 'SonataFlow Alert'
    text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
```

# Troubleshooting

## Common Issues

### Logs Still in Plain Text Format

**Problem**: Logs appear as traditional text instead of JSON after configuration.

**Solutions**:
1. Verify `quarkus-logging-json` extension is in your workflow's `pom.xml`
2. Rebuild and redeploy the workflow container image
3. Check that `quarkus.log.console.json=true` is correctly set in ConfigMap
4. Restart the workflow pod after ConfigMap changes

### No Process Instance Context in Logs

**Problem**: JSON logs work but `processInstanceId` is always empty or missing.

**Solutions**:
1. Verify workflow instances are actually running (check workflow status)
2. Ensure `quarkus.log.console.json.print-details=true` is set
3. Check if your SonataFlow version automatically populates MDC (may require custom implementation)

### Promtail Not Collecting Logs

**Problem**: Loki shows no data from workflow pods.

**Solutions**:
1. Verify pod label selector in Promtail configuration matches your workflow pods
2. Check Promtail logs: `oc logs -l app=promtail`
3. Ensure Promtail has proper RBAC permissions to read pod logs
4. Verify namespace configuration in `scrape_configs`

### High Resource Usage

**Problem**: JSON logging causes performance issues.

**Solutions**:
1. Adjust log levels to reduce volume: `quarkus.log.category."org.kie.kogito".level=WARN`
2. Configure log rotation and retention policies
3. Use asynchronous logging: `quarkus.log.console.async=true`
4. Monitor storage and network bandwidth usage

# Notes and Considerations

• **Resource Planning**: JSON logging increases log volume by 20-30% compared to plain text
• **Retention Policies**: Configure appropriate log retention (typically 30-90 days for workflow logs)
• **Index Optimization**: Use time-based indices for better performance with large log volumes
• **Security**: Ensure log aggregation respects namespace isolation and RBAC policies
• **Backup Strategy**: Include log indices in disaster recovery planning
• **Cost Management**: Monitor storage costs, especially with high-throughput workflows

## Additional Resources

• **[Complete PLG Observability Guide](../observability/)**: Comprehensive documentation with production-ready configurations
• **[Quick Start Guide](../observability/quick-start.md)**: 15-minute deployment walkthrough
• **[YAML Manifests](../observability/deployment-manifests.md)**: All Kubernetes resources for manual deployment
• **[Advanced Promtail Configuration](../observability/promtail-config.md)**: Detailed log parsing and collection setup

For detailed setup instructions and advanced configurations, refer to the comprehensive observability documentation and OpenShift logging best practices.
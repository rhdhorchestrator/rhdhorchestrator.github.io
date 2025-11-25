---
title: "Configure OpenTelemetry for SonataFlow Workflows"
date: 2025-11-25
---

# Configure OpenTelemetry for SonataFlow Workflows

This document provides comprehensive guidance for enabling and configuring OpenTelemetry observability in SonataFlow workflows deployed on OpenShift/Kubernetes clusters. You'll learn how to configure your workflows for distributed tracing, metrics collection, and log aggregation using industry-standard observability tools.

## Prerequisites

* SonataFlow Operator installed and configured
* Kubernetes/OpenShift cluster with appropriate resources
* Access to deploy observability infrastructure
* Basic understanding of OpenTelemetry concepts

## Overview

The OpenTelemetry integration for SonataFlow provides:

- **Distributed Tracing**: Track workflow execution across multiple services and steps
- **Metrics Collection**: Monitor performance, duration, and success rates
- **Log Aggregation**: Centralized logging with trace correlation
- **Context Propagation**: Maintain trace context across workflow boundaries and async operations

> **Note**: The OpenTelemetry feature is available in SonataFlow runtime through the `sonataflow-quarkus-otel-extension` extension, which provides comprehensive observability capabilities with minimal configuration overhead.

## Architecture Overview

SonataFlow workflows emit telemetry data using the OpenTelemetry Protocol (OTLP) to collectors or directly to observability platforms:

```
┌──────────────────────┐
│ SonataFlow Workflows │
│  (Quarkus + OTEL)    │
└─────────┬────────────┘
          │ OTLP (gRPC/HTTP)
          │
    ┌─────┴─────┐
    │           │
    │ Traces    │ Logs
    │ :4317     │ :3100/otlp
    ▼           ▼
┌─────────┐ ┌─────────┐
│ Jaeger  │ │  Loki   │
│(Tracing)│ │ (Logs)  │
└─────────┘ └─────────┘
    │           │
    └─────┬─────┘
          │
          ▼
    ┌─────────┐
    │ Grafana │
    │(Visualize)│
    └─────────┘

Optional: OpenTelemetry Collector
for advanced processing, filtering,
and multi-backend export
```

## Configuration

### Enable OpenTelemetry Extension

To enable OpenTelemetry in your SonataFlow workflow, add the extension to your project:

**1. Add Extension Dependency**

Add the Sonataflow Open Telemetry extension in the `QUARKUS_EXTENSIONS` environment variable when building the workflow image:

```
export QUARKUS_EXTENSIONS="${QUARKUS_EXTENSIONS},org.apache.kie.sonataflow:sonataflow-quarkus-otel-extension"
```


**2. Configure Workflow Properties**

The following properties must be configured in the ConfigMap that holds your workflow's `application.properties`. When a SonataFlow CR is deployed, it automatically generates a `{workflow-name}-props` ConfigMap that contains these properties.

### Basic OpenTelemetry Configuration

Add the following properties to your workflow's `application.properties`:

```properties
# Application Identity
quarkus.application.name=my-workflow
quarkus.application.version=1.0.0

# OpenTelemetry Configuration
quarkus.otel.enabled=true
quarkus.otel.traces.enabled=true
quarkus.otel.metrics.enabled=true
quarkus.otel.logs.enabled=true

# Service Resource Attributes
quarkus.otel.resource.attributes=\
  service.name=my-workflow,\
  service.namespace=workflows,\
  service.version=1.0.0,\
  deployment.environment=production

# SonataFlow Specific Configuration
sonataflow.otel.enabled=true
sonataflow.otel.serviceName=${quarkus.application.name:sonataflow-workflow-service}
sonataflow.otel.serviceVersion=${quarkus.application.version:unknown}
sonataflow.otel.spans.enabled=true
sonataflow.otel.events.enabled=true

# Context Propagation
quarkus.otel.propagators=tracecontext,baggage,jaeger

# Instrumentation
quarkus.datasource.jdbc.telemetry=true
quarkus.otel.instrument.rest=true
quarkus.otel.instrument.grpc=true
```

### Exporter Configuration

Choose one of the following export strategies based on your observability platform:

#### Option 1: OTLP Exporter (Recommended)

```properties
# OTLP Exporter - Direct to Jaeger
quarkus.otel.exporter.otlp.endpoint=http://jaeger-collector.observability.svc.cluster.local:4317
quarkus.otel.exporter.otlp.protocol=grpc
quarkus.otel.traces.exporter=cdi

# Batch Processing for Production
quarkus.otel.bsp.schedule.delay=5s
quarkus.otel.bsp.max.export.batch.size=512
quarkus.otel.bsp.export.timeout=2s
quarkus.otel.bsp.max.queue.size=2048
```

#### Option 2: Direct Export to External Platform

```properties
# Example: Direct export to Jaeger
quarkus.otel.exporter.otlp.endpoint=http://jaeger-collector:4317
quarkus.otel.exporter.otlp.protocol=grpc
quarkus.otel.traces.exporter=cdi
```


### Externalized Configuration

For production deployments, use environment variables to externalize configuration:

```properties
# Externalized Configuration
quarkus.otel.exporter.otlp.endpoint=${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4317}
quarkus.otel.exporter.otlp.headers=${OTEL_EXPORTER_OTLP_HEADERS:}
quarkus.application.name=${OTEL_SERVICE_NAME:my-workflow}
quarkus.otel.resource.attributes=${OTEL_RESOURCE_ATTRIBUTES:deployment.environment=dev}
```

## Observability Tools Configuration

This section provides working examples for two essential observability tools that integrate seamlessly with SonataFlow's OpenTelemetry implementation.

### Tool 1: Jaeger Distributed Tracing

Jaeger provides distributed tracing visualization for SonataFlow workflows.

#### Deploy Jaeger on OpenShift/Kubernetes

**All-in-One Deployment (Development/Testing):**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jaeger-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: jaeger-system
  labels:
    app: jaeger
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:1.59
        env:
        - name: COLLECTOR_OTLP_ENABLED
          value: "true"
        ports:
        - containerPort: 16686
          name: query
        - containerPort: 4317
          name: otlp-grpc
        - containerPort: 4318
          name: otlp-http
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /
            port: 14269
          initialDelaySeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 14269
          initialDelaySeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-collector
  namespace: jaeger-system
  labels:
    app: jaeger
spec:
  selector:
    app: jaeger
  ports:
  - name: otlp-grpc
    port: 4317
    targetPort: 4317
  - name: otlp-http
    port: 4318
    targetPort: 4318
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-query
  namespace: jaeger-system
  labels:
    app: jaeger
spec:
  selector:
    app: jaeger
  ports:
  - name: query-http
    port: 16686
    targetPort: 16686
  type: ClusterIP
```

**OpenShift Route for UI Access:**

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: jaeger-query
  namespace: jaeger-system
spec:
  to:
    kind: Service
    name: jaeger-query
  port:
    targetPort: query-http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

**Workflow Configuration for Jaeger:**

```properties
# Direct connection to Jaeger
quarkus.otel.exporter.otlp.endpoint=http://jaeger-collector.jaeger-system.svc.cluster.local:4317
quarkus.otel.exporter.otlp.protocol=grpc
quarkus.otel.traces.exporter=cdi

# Additional Jaeger-specific propagation
quarkus.otel.propagators=tracecontext,baggage,jaeger
```

**Production Deployment with Elasticsearch:**

For production environments, use the Jaeger Operator with Elasticsearch storage:

```yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
  namespace: observability
spec:
  strategy: production
  storage:
    type: elasticsearch
    elasticsearch:
      nodeCount: 3
      storage:
        storageClassName: gp3
        size: 50Gi
      resources:
        requests:
          cpu: 500m
          memory: 4Gi
        limits:
          cpu: 1000m
          memory: 8Gi
  collector:
    replicas: 2
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

### Tool 2: Loki Log Aggregation

Grafana Loki provides scalable log aggregation with native OpenTelemetry support, enabling comprehensive log analysis and correlation with traces.

Loki natively supports OpenTelemetry Protocol (OTLP) for direct log ingestion from SonataFlow workflows.

#### Deploy Loki on OpenShift/Kubernetes

**Loki Configuration for OpenTelemetry:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-config
  namespace: observability
data:
  loki-config.yaml: |
    auth_enabled: false

    server:
      http_listen_port: 3100
      grpc_listen_port: 9096

    common:
      path_prefix: /loki
      storage:
        filesystem:
          chunks_directory: /loki/chunks
          rules_directory: /loki/rules
      replication_factor: 1
      ring:
        instance_addr: 127.0.0.1
        kvstore:
          store: inmemory

    distributor:
      otlp_config:
        # Default resource attributes as index labels
        default_resource_attributes_as_index_labels:
          - service.name
          - service.namespace
          - deployment.environment
          - k8s.namespace.name
          - k8s.cluster.name

    limits_config:
      # Enable structured metadata (default in Loki 3.0+)
      allow_structured_metadata: true
      # Maximum number of index labels per stream
      max_label_names_per_series: 15

    schema_config:
      configs:
        - from: 2024-01-01
          store: tsdb
          object_store: filesystem
          schema: v13  # Required for OTLP support
          index:
            prefix: index_
            period: 24h
```

**Loki Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loki
  namespace: observability
  labels:
    app: loki
spec:
  replicas: 1
  selector:
    matchLabels:
      app: loki
  template:
    metadata:
      labels:
        app: loki
    spec:
      securityContext:
        fsGroup: 10001
        runAsUser: 10001
        runAsNonRoot: true
      containers:
      - name: loki
        image: grafana/loki:3.0.0
        args:
          - -config.file=/etc/loki/loki-config.yaml
        ports:
        - containerPort: 3100
          name: http-metrics
        - containerPort: 9096
          name: grpc
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
        volumeMounts:
        - name: config
          mountPath: /etc/loki
        - name: storage
          mountPath: /loki
        livenessProbe:
          httpGet:
            path: /ready
            port: 3100
          initialDelaySeconds: 45
        readinessProbe:
          httpGet:
            path: /ready
            port: 3100
          initialDelaySeconds: 45
      volumes:
      - name: config
        configMap:
          name: loki-config
      - name: storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: loki
  namespace: observability
  labels:
    app: loki
spec:
  selector:
    app: loki
  ports:
  - name: http-metrics
    port: 3100
    targetPort: 3100
  - name: grpc
    port: 9096
    targetPort: 9096
  type: ClusterIP
```

#### Workflow Configuration for Loki

**Direct Connection to Loki (Recommended for simplicity):**

```properties
# OpenTelemetry Configuration
quarkus.otel.enabled=true
quarkus.otel.traces.enabled=true
quarkus.otel.metrics.enabled=true
quarkus.otel.logs.enabled=true

# OTLP Exporter - Send logs to Loki, traces to Jaeger
quarkus.otel.exporter.otlp.logs.endpoint=http://loki.observability.svc.cluster.local:3100/otlp
quarkus.otel.exporter.otlp.traces.endpoint=http://jaeger-collector.observability.svc.cluster.local:4317
quarkus.otel.exporter.otlp.protocol=grpc

# JSON Logging for better structure
quarkus.log.console.json=true
quarkus.log.console.json.pretty-print=false

# Include trace correlation in logs
quarkus.log.console.format=%d{yyyy-MM-dd HH:mm:ss,SSS} %-5p [%c{3.}] (%t) traceId=%X{traceId}, spanId=%X{spanId} %s%e%n

# Resource attributes for Loki labels
quarkus.otel.resource.attributes=\
  service.name=greeting-workflow,\
  service.namespace=workflows,\
  deployment.environment=production
```

### Optional: OpenTelemetry Collector for Advanced Processing

For production environments requiring log processing, enrichment, or multi-backend export, you can optionally deploy an OpenTelemetry Collector between your workflows and the observability backends.

**Benefits of using a Collector:**
- Log enrichment with Kubernetes metadata
- Filtering and sampling
- Multi-destination export (e.g., logs to both Loki and external SIEM)
- Centralized processing and transformation

**Quick Collector Setup:**

```properties
# Change workflow configuration to send to collector instead
quarkus.otel.exporter.otlp.endpoint=http://otel-collector.observability.svc.cluster.local:4317
```

**Collector Configuration:**
```yaml
# Collector routes to both Jaeger and Loki
exporters:
  otlp/jaeger:
    endpoint: jaeger-collector:4317
  otlphttp/loki:
    endpoint: http://loki:3100/otlp

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/loki]
```

> For complete collector deployment manifests, see the [OpenTelemetry Collector documentation](https://opentelemetry.io/docs/collector/deployment/).

## Generated Telemetry Data

When your SonataFlow workflow runs with OpenTelemetry enabled, it generates comprehensive observability data:

### Trace Spans

SonataFlow automatically creates spans for:

- **Process execution** - Overall workflow span with process ID and version
- **Node execution** - Individual spans for each workflow node/state
- **Function calls** - HTTP calls to external services with full context
- **Error handling** - Spans marked with error status and details

Example span attributes generated:
```
sonataflow.process.instance.id: greeting-abc123-456-789
sonataflow.process.id: greeting
sonataflow.process.version: 1.0.0
sonataflow.process.instance.state: ACTIVE
sonataflow.process.instance.node: ChooseOnLanguage
service.name: greeting-workflow
service.namespace: workflows
```

### Events and Logs

Process events are automatically added:
- `process.instance.start` - When workflow begins
- `NODE_STARTED` - When each node begins execution
- `NODE_COMPLETED` - When each node finishes
- Log events with trace correlation

### Metrics

Metrics are exported for:
- Process duration and success rates
- Node execution times
- Error rates and types
- Resource utilization

## What's More: Visualization and Monitoring

Once your telemetry data is flowing, you can leverage various visualization and monitoring approaches:

### Jaeger UI Exploration

Access the Jaeger UI to:
- **Trace Search**: Find traces by service, operation, or time range
- **Service Map**: Visualize service dependencies and call patterns
- **Performance Analysis**: Identify bottlenecks and latency issues
- **Error Investigation**: Trace error propagation through workflow steps

Example Jaeger query for your workflow:
```
service:greeting-workflow operation:workflow.execute
```

### Grafana Dashboards

Create dashboards to monitor:
- **Workflow Success Rate**: Percentage of successful executions
- **Execution Duration**: P50, P95, P99 latencies by workflow and node
- **Error Rate Trends**: Track error patterns over time
- **Resource Utilization**: Monitor workflow resource consumption

### Loki Log Exploration

**Grafana Explore Queries for SonataFlow logs:**

```logql
# All logs from greeting workflow
{service_name="greeting-workflow"}

# Error logs only
{service_name="greeting-workflow"} |= "ERROR"

# Logs for specific workflow node
{service_name="greeting-workflow"} | json | node_name="ChooseOnLanguage"

# Logs with trace ID correlation
{service_name="greeting-workflow"} | json | trace_id != "" | line_format "{{.trace_id}}: {{.message}}"

# Count workflow executions by language choice
sum by (language) (count_over_time({service_name="greeting-workflow"} |~ "language.*English|Spanish" [5m]))

# Error rate by workflow
sum by (service_name) (rate({service_namespace="workflows"} |= "ERROR" [5m])) /
sum by (service_name) (rate({service_namespace="workflows"} [5m]))

# Workflow execution duration analysis
{service_name="greeting-workflow"}
  | json
  | duration_ms != ""
  | line_format "Duration: {{.duration_ms}}ms for {{.workflow_id}}"

# Correlation between traces and logs
{service_name="greeting-workflow"}
  | json
  | trace_id="your-trace-id-here"
  | line_format "{{.timestamp}} [{{.level}}] {{.message}}"
```

**Example Loki Dashboard Panels:**

1. **Log Volume Timeline**
   ```logql
   sum by (service_name) (count_over_time({service_namespace="workflows"}[1m]))
   ```

2. **Error Logs Table**
   ```logql
   {service_namespace="workflows"} |= "ERROR" | json
   ```

3. **Workflow Success vs Failure**
   ```logql
   # Success
   sum(count_over_time({service_name="greeting-workflow"} |~ "completed successfully" [5m]))

   # Failures
   sum(count_over_time({service_name="greeting-workflow"} |= "ERROR" [5m]))
   ```

4. **Language Choice Distribution (Greeting-specific)**
   ```logql
   sum by (language) (count_over_time({service_name="greeting-workflow"} |~ "English|Spanish" [1h]))
   ```

### Alerting

Set up alerts for:
- High error rates in critical workflow steps
- Unusual execution duration increases
- Service dependency failures
- Resource exhaustion patterns

### Log Correlation

Correlate logs using trace IDs:
- Search logs by trace ID to see complete execution context
- Track data flow through complex workflow states
- Debug issues with full observability context

### Advanced Analysis

Use the collected data for:
- **Performance optimization** - Identify slow workflow nodes
- **Capacity planning** - Understand resource usage patterns
- **Business insights** - Track workflow completion rates and user experience
- **Troubleshooting** - Root cause analysis with full trace context

## Troubleshooting

### Common Issues

#### No Traces Appearing

**Check workflow configuration:**
```bash
# Verify OpenTelemetry is enabled
kubectl get cm onboarding-workflow-props -n workflows -o yaml

# Check pod logs for OpenTelemetry initialization
kubectl logs -n workflows deployment/onboarding-workflow | grep -i "otel\|trace"
```

**Verify connectivity:**
```bash
# Test Jaeger endpoint from workflow pod
kubectl exec -n workflows deployment/greeting -- curl -v http://jaeger-collector.observability.svc.cluster.local:4317
```

#### Authentication Issues

**For platforms requiring authentication, configure headers:**
```properties
quarkus.otel.exporter.otlp.headers=authorization=Bearer ${API_TOKEN}
```

**Test authentication:**
```bash
curl -v -X POST \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/x-protobuf" \
  https://your-endpoint.com/v1/traces
```

#### High Memory Usage

**Configure memory limits in collector:**
```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 1000
    spike_limit_mib: 200
```

#### Context Lost Between Steps

**Ensure proper propagation:**
```properties
# Include all required propagators
quarkus.otel.propagators=tracecontext,baggage,jaeger

# Enable JSON logging to verify trace IDs
quarkus.log.console.json=true
```

### Verification Steps

**1. Check OpenTelemetry extension is loaded:**
```bash
kubectl logs -n workflows deployment/onboarding-workflow | grep "sonataflow-quarkus-otel"
```

**2. Verify trace export:**
```bash
# Look for successful trace exports
kubectl logs -n workflows deployment/greeting | grep -i "export\|batch"
```

**3. Check Jaeger health:**
```bash
# Verify Jaeger is receiving data
kubectl logs -n observability deployment/jaeger | grep -i "span\|trace"
```

**4. Test with debug exporter:**
```properties
# Temporarily enable console logging
%dev.quarkus.otel.traces.exporter=logging
```

## Additional Resources

* [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
* [Quarkus OpenTelemetry Guide](https://quarkus.io/guides/opentelemetry)
* [Jaeger Documentation](https://www.jaegertracing.io/docs/)
* [OpenTelemetry Collector Configuration](https://opentelemetry.io/docs/collector/configuration/)
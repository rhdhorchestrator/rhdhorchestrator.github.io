---
title: Promtail Configuration
date: "2025-11-24"
weight: 2
---

# Advanced Promtail Configuration for SonataFlow

This page provides detailed Promtail configuration examples for scraping, parsing, and forwarding SonataFlow workflow logs to Loki.

## Overview

Promtail is configured to:
1. Discover SonataFlow workflow pods automatically using Kubernetes service discovery
2. Parse JSON logs and extract MDC context fields
3. Add Kubernetes metadata as labels
4. Forward structured logs to Loki

## Standalone Promtail Configuration

If you prefer to deploy Promtail separately (not via Helm), use this configuration.

### ConfigMap

Save as `promtail-config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: sonataflow-observability
  labels:
    app: promtail
data:
  promtail.yaml: |
    # Promtail server configuration
    server:
      http_listen_port: 3101
      grpc_listen_port: 0
      log_level: info
      log_format: json

    # Position tracking file
    positions:
      filename: /run/promtail/positions.yaml

    # Loki client configuration
    clients:
      - url: http://loki.sonataflow-observability.svc.cluster.local:3100/loki/api/v1/push
        tenant_id: ""

        # Batching configuration
        batchwait: 1s
        batchsize: 1048576  # 1MB

        # Backoff configuration for retries
        backoff_config:
          min_period: 500ms
          max_period: 5m
          max_retries: 10

        # Request timeout
        timeout: 10s

        # External labels (applied to all logs)
        external_labels:
          cluster: openshift
          environment: production

    # Target configuration
    target_config:
      sync_period: 10s

    # ==============================================================================
    # Scrape Configurations
    # ==============================================================================
    scrape_configs:

      # ==========================================================================
      # Job: SonataFlow Workflow Pods
      # ==========================================================================
      - job_name: sonataflow-workflows

        # Kubernetes service discovery
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - sonataflow-infra
                - default
                # Add additional namespaces here

        # Pipeline stages for log processing
        pipeline_stages:

          # Stage 1: Extract container logs from Docker JSON
          - docker: {}

          # Stage 2: Parse JSON log format
          - json:
              expressions:
                # Top-level fields
                timestamp: timestamp
                level: level
                logger: logger
                message: message
                thread: thread
                exception: exception
                stackTrace: stackTrace

                # MDC fields (Mapped Diagnostic Context)
                processInstanceId: MDC.processInstanceId
                processId: MDC.processId
                traceId: MDC.traceId
                spanId: MDC.spanId
                workflowId: MDC.workflowId
                workflowVersion: MDC.workflowVersion
                state: MDC.state
                nodeId: MDC.nodeId
                userId: MDC.userId
                businessKey: MDC.businessKey
                correlationId: MDC.correlationId

                # Custom fields
                eventType: eventType
                duration: duration

          # Stage 3: Parse timestamp
          - timestamp:
              source: timestamp
              format: RFC3339Nano
              fallback_formats:
                - "2006-01-02T15:04:05.999999999Z07:00"
                - "2006-01-02T15:04:05.999Z"
                - "2006-01-02T15:04:05Z"
                - Unix

          # Stage 4: Extract and set log level label
          - labels:
              level:

          # Stage 5: Extract logger name
          - labels:
              logger:

          # Stage 6: Extract MDC fields as labels for efficient querying
          - labels:
              processInstanceId:
              processId:
              traceId:
              spanId:
              workflowId:
              state:

          # Stage 7: Add structured metadata (searchable but not indexed)
          - structured_metadata:
              thread:
              userId:
              businessKey:
              correlationId:
              workflowVersion:
              nodeId:
              eventType:

          # Stage 8: Extract metrics (optional)
          - metrics:
              # Count log entries by level
              log_lines_total:
                type: Counter
                description: "Total log lines processed"
                source: level
                config:
                  action: inc

              # Track workflow execution duration
              workflow_duration_seconds:
                type: Histogram
                description: "Workflow execution duration"
                source: duration
                config:
                  buckets: [0.1, 0.5, 1, 5, 10, 30, 60, 300]

          # Stage 9: Drop noisy logs (optional)
          - match:
              selector: '{level="DEBUG"}'
              stages:
                - drop:
                    expression: ".*health check.*"
                    drop_counter_reason: "health_check"

          # Stage 10: Enrich error logs with exception details
          - match:
              selector: '{level=~"ERROR|FATAL"}'
              stages:
                - structured_metadata:
                    exception:
                    stackTrace:

          # Stage 11: Format output message
          - output:
              source: message

        # Relabeling configuration for Kubernetes metadata
        relabel_configs:

          # Keep only pods with SonataFlow workflow label
          - source_labels: [__meta_kubernetes_pod_label_sonataflow_org_workflow_app]
            action: keep
            regex: (.+)

          # Add namespace
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
            action: replace

          # Add pod name
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
            action: replace

          # Add container name
          - source_labels: [__meta_kubernetes_pod_container_name]
            target_label: container
            action: replace

          # Add workflow name from label
          - source_labels: [__meta_kubernetes_pod_label_sonataflow_org_workflow_app]
            target_label: workflow_name
            action: replace

          # Add workflow version from annotation
          - source_labels: [__meta_kubernetes_pod_annotation_sonataflow_org_version]
            target_label: workflow_version
            action: replace

          # Add pod phase
          - source_labels: [__meta_kubernetes_pod_phase]
            target_label: pod_phase
            action: replace

          # Add node name
          - source_labels: [__meta_kubernetes_pod_node_name]
            target_label: node
            action: replace

          # Add pod IP
          - source_labels: [__meta_kubernetes_pod_ip]
            target_label: pod_ip
            action: replace

          # Add pod UID for uniqueness
          - source_labels: [__meta_kubernetes_pod_uid]
            target_label: pod_uid
            action: replace

          # Add controller name (Deployment/StatefulSet)
          - source_labels: [__meta_kubernetes_pod_controller_name]
            target_label: controller
            action: replace

          # Add custom app label
          - source_labels: [__meta_kubernetes_pod_label_app]
            target_label: app
            action: replace

          # Add environment label
          - source_labels: [__meta_kubernetes_pod_label_environment]
            target_label: environment
            action: replace

          # Add team label (for multi-tenancy)
          - source_labels: [__meta_kubernetes_pod_label_team]
            target_label: team
            action: replace

          # Set the log file path
          - source_labels: [__meta_kubernetes_pod_uid, __meta_kubernetes_pod_container_name]
            target_label: __path__
            separator: /
            replacement: /var/log/pods/*$1/*/*.log

      # ==========================================================================
      # Job: SonataFlow Platform Services (Data Index, Job Service)
      # ==========================================================================
      - job_name: sonataflow-platform-services

        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - sonataflow-infra

        pipeline_stages:
          - docker: {}

          - json:
              expressions:
                timestamp: timestamp
                level: level
                logger: logger
                message: message
                thread: thread
                exception: exception
                serviceType: MDC.serviceType
                queryId: MDC.queryId
                jobId: MDC.jobId

          - timestamp:
              source: timestamp
              format: RFC3339Nano

          - labels:
              level:
              logger:
              serviceType:

          - structured_metadata:
              thread:
              queryId:
              jobId:

          - output:
              source: message

        relabel_configs:
          # Match only Data Index and Job Service
          - source_labels: [__meta_kubernetes_pod_label_sonataflow_org_service]
            action: keep
            regex: (sonataflow-platform-data-index-service|sonataflow-platform-jobs-service)

          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace

          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod

          - source_labels: [__meta_kubernetes_pod_container_name]
            target_label: container

          - source_labels: [__meta_kubernetes_pod_label_sonataflow_org_service]
            target_label: service_name

          - source_labels: [__meta_kubernetes_pod_node_name]
            target_label: node

          - source_labels: [__meta_kubernetes_pod_uid, __meta_kubernetes_pod_container_name]
            target_label: __path__
            separator: /
            replacement: /var/log/pods/*$1/*/*.log

      # ==========================================================================
      # Job: SonataFlow Operator Logs
      # ==========================================================================
      - job_name: sonataflow-operator

        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - sonataflow-operator-system

        pipeline_stages:
          - docker: {}

          - json:
              expressions:
                timestamp: ts
                level: level
                logger: logger
                message: msg
                error: error
                reconcileRequest: reconcileRequest

          - timestamp:
              source: timestamp
              format: Unix

          - labels:
              level:
              logger:

          - structured_metadata:
              error:
              reconcileRequest:

          - output:
              source: message

        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
            action: keep
            regex: sonataflow-operator

          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace

          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod

          - source_labels: [__meta_kubernetes_pod_container_name]
            target_label: container

          - source_labels: [__meta_kubernetes_pod_uid, __meta_kubernetes_pod_container_name]
            target_label: __path__
            separator: /
            replacement: /var/log/pods/*$1/*/*.log

      # ==========================================================================
      # Job: PostgreSQL Logs (if using for persistence)
      # ==========================================================================
      - job_name: postgresql

        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - sonataflow-infra

        pipeline_stages:
          - docker: {}

          # PostgreSQL logs are usually plain text
          - regex:
              expression: '^(?P<timestamp>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d{3} \w+) \[(?P<pid>\d+)\] (?P<level>\w+):  (?P<message>.*)'

          - timestamp:
              source: timestamp
              format: "2006-01-02 15:04:05.000 MST"

          - labels:
              level:

          - structured_metadata:
              pid:

          - output:
              source: message

        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            action: keep
            regex: postgres.*

          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace

          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod

          - source_labels: [__meta_kubernetes_pod_container_name]
            target_label: container

          - source_labels: [__meta_kubernetes_pod_uid, __meta_kubernetes_pod_container_name]
            target_label: __path__
            separator: /
            replacement: /var/log/pods/*$1/*/*.log
```

## DaemonSet Deployment

Deploy Promtail as a DaemonSet to collect logs from all nodes:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
  namespace: sonataflow-observability
  labels:
    app: promtail
spec:
  selector:
    matchLabels:
      app: promtail
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: promtail
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3101"
    spec:
      serviceAccountName: promtail

      # Security context
      securityContext:
        runAsUser: 0
        runAsGroup: 0
        fsGroup: 0

      # Host networking for log access
      hostNetwork: false

      # DNS policy
      dnsPolicy: ClusterFirst

      # Tolerations to run on all nodes
      tolerations:
        - effect: NoSchedule
          operator: Exists
        - effect: NoExecute
          operator: Exists

      containers:
        - name: promtail
          image: grafana/promtail:2.9.3
          imagePullPolicy: IfNotPresent

          args:
            - -config.file=/etc/promtail/promtail.yaml
            - -config.expand-env=true

          # Environment variables
          env:
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName

          # Ports
          ports:
            - name: http-metrics
              containerPort: 3101
              protocol: TCP

          # Security context
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: false
            runAsUser: 0

          # Liveness probe
          livenessProbe:
            httpGet:
              path: /ready
              port: http-metrics
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 1
            successThreshold: 1
            failureThreshold: 5

          # Readiness probe
          readinessProbe:
            httpGet:
              path: /ready
              port: http-metrics
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 1
            successThreshold: 1
            failureThreshold: 5

          # Resources
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi

          # Volume mounts
          volumeMounts:
            - name: config
              mountPath: /etc/promtail
            - name: run
              mountPath: /run/promtail
            - name: pods
              mountPath: /var/log/pods
              readOnly: true
            - name: docker
              mountPath: /var/lib/docker/containers
              readOnly: true

      # Volumes
      volumes:
        - name: config
          configMap:
            name: promtail-config
        - name: run
          emptyDir: {}
        - name: pods
          hostPath:
            path: /var/log/pods
            type: Directory
        - name: docker
          hostPath:
            path: /var/lib/docker/containers
            type: Directory
```

## Pipeline Stage Examples

### Example 1: Multi-line Exception Handling

```yaml
pipeline_stages:
  - multiline:
      firstline: '^\d{4}-\d{2}-\d{2}'
      max_wait_time: 3s
      max_lines: 1000

  - json:
      expressions:
        timestamp: timestamp
        level: level
        message: message
        exception: exception
        stackTrace: stackTrace

  - match:
      selector: '{level="ERROR"}'
      stages:
        - multiline:
            firstline: '^[^\s]'
        - output:
            source: message
```

### Example 2: Redacting Sensitive Information

```yaml
pipeline_stages:
  - json:
      expressions:
        message: message

  # Redact passwords
  - replace:
      expression: '(password|pwd|secret)=\S+'
      replace: '$1=***REDACTED***'

  # Redact tokens
  - replace:
      expression: '(token|api_key):\s*"[^"]+"'
      replace: '$1: "***REDACTED***"'

  # Redact credit cards
  - replace:
      expression: '\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b'
      replace: '****-****-****-****'
```

### Example 3: Custom Label Extraction

```yaml
pipeline_stages:
  - json:
      expressions:
        message: message
        customField: custom.nested.field

  # Extract labels from message using regex
  - regex:
      expression: 'workflow=(?P<workflow_name>\w+)'
      source: message

  - labels:
      workflow_name:

  # Template to create new label
  - template:
      source: workflow_name
      template: 'workflow_{{ .Value }}'

  - labels:
      templated_name:
```

### Example 4: Conditional Processing

```yaml
pipeline_stages:
  - json:
      expressions:
        level: level
        message: message
        processInstanceId: MDC.processInstanceId

  # Process ERROR logs differently
  - match:
      selector: '{level="ERROR"}'
      stages:
        - structured_metadata:
            severity: "high"
        - metrics:
            error_count:
              type: Counter
              description: "Total error count"
              config:
                action: inc

  # Drop DEBUG logs in production
  - match:
      selector: '{environment="production", level="DEBUG"}'
      action: drop

  # Enrich workflow completion logs
  - match:
      selector: '{message=~".*workflow completed.*"}'
      stages:
        - structured_metadata:
            status: "completed"
        - metrics:
            workflow_completions:
              type: Counter
              description: "Workflow completions"
              config:
                action: inc
```

## Testing Promtail Configuration

### 1. Validate Configuration

```bash
# Download Promtail binary
curl -O -L "https://github.com/grafana/loki/releases/download/v2.9.3/promtail-linux-amd64.zip"
unzip promtail-linux-amd64.zip
chmod +x promtail-linux-amd64

# Test configuration
./promtail-linux-amd64 -config.file=promtail.yaml -dry-run
```

### 2. Test with Sample Logs

Create a test log file:

```bash
cat > test-logs.json << 'EOF'
{"timestamp":"2025-11-24T10:30:45.123Z","level":"INFO","logger":"org.kie.kogito","message":"Workflow started","MDC":{"processInstanceId":"abc-123","traceId":"trace-456","workflowId":"onboarding"}}
{"timestamp":"2025-11-24T10:30:46.456Z","level":"ERROR","logger":"org.kie.kogito","message":"Failed to complete step","MDC":{"processInstanceId":"abc-123","state":"ApprovalStep"},"exception":"NullPointerException"}
EOF
```

Configure Promtail to read from file:

```yaml
scrape_configs:
  - job_name: test
    static_configs:
      - targets:
          - localhost
        labels:
          job: test
          __path__: /path/to/test-logs.json

    pipeline_stages:
      - json:
          expressions:
            timestamp: timestamp
            level: level
            message: message
            processInstanceId: MDC.processInstanceId
      - labels:
          level:
          processInstanceId:
      - output:
          source: message
```

### 3. Check Promtail Metrics

```bash
# Port-forward Promtail
oc port-forward -n sonataflow-observability ds/promtail 3101:3101

# Check metrics
curl http://localhost:3101/metrics
```

Look for:
- `promtail_targets_active_total` - Number of active targets
- `promtail_read_lines_total` - Lines read
- `promtail_sent_entries_total` - Entries sent to Loki
- `promtail_dropped_entries_total` - Dropped entries

## Common Issues and Solutions

### Issue 1: Logs Not Being Scraped

**Symptoms**: No logs appearing in Loki

**Solutions**:

1. Check pod labels match selector:
```bash
oc get pods -n sonataflow-infra --show-labels
```

2. Verify log path exists:
```bash
oc exec -it promtail-xxxxx -n sonataflow-observability -- ls -la /var/log/pods/
```

3. Check Promtail targets:
```bash
curl http://localhost:3101/targets
```

### Issue 2: JSON Parsing Failures

**Symptoms**: Logs appear as plain text in Loki

**Solutions**:

1. Verify JSON format:
```bash
oc logs -n sonataflow-infra <workflow-pod> | jq .
```

2. Add debug stage:
```yaml
pipeline_stages:
  - json:
      expressions:
        message: message
  - output:
      source: message
```

3. Check for Docker wrapper:
```yaml
pipeline_stages:
  - docker: {}  # This must come first!
  - json:
      expressions:
        message: message
```

### Issue 3: High Memory Usage

**Symptoms**: Promtail consuming too much memory

**Solutions**:

1. Reduce batch size:
```yaml
clients:
  - batchsize: 524288  # 512KB instead of 1MB
```

2. Limit targets:
```yaml
target_config:
  sync_period: 30s  # Slower sync
```

3. Drop more logs:
```yaml
pipeline_stages:
  - match:
      selector: '{level="DEBUG"}'
      action: drop
```

## Advanced Features

### Distributed Tracing Integration

Link logs to traces in Tempo:

```yaml
pipeline_stages:
  - json:
      expressions:
        traceId: MDC.traceId

  - tenant:
      value: default

  - structured_metadata:
      traceID:  # Note: capital 'ID' for Tempo
```

### Dynamic Labels

Create labels based on log content:

```yaml
pipeline_stages:
  - json:
      expressions:
        message: message

  - regex:
      expression: '^(?P<verb>GET|POST|PUT|DELETE)'
      source: message

  - labels:
      http_verb:
```

### Rate Limiting

Limit logs sent to Loki:

```yaml
clients:
  - url: http://loki:3100/loki/api/v1/push
    backoff_config:
      max_period: 1m
      max_retries: 5

    # Rate limiting
    tenant_id: ""
    batchwait: 5s  # Wait longer before sending
```

## Next Steps

- [View logs in Grafana Dashboard](grafana-dashboard.md)
- [Learn LogQL Queries](logql-queries.md)
- [Deploy with RBAC](deployment-manifests.md)

---
title: Helm Values Configuration
date: "2025-11-24"
weight: 1
---

# Grafana Loki Stack Helm Values for OpenShift

This page provides a complete, production-ready Helm values file for deploying the Grafana Loki stack on OpenShift to monitor SonataFlow workflows.

## Installation

```bash
# Add the Grafana Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create the namespace
oc create namespace sonataflow-observability

# Deploy the Loki stack
helm install loki-stack grafana/loki-stack \
  --namespace sonataflow-observability \
  --values loki-stack-values.yaml
```

## Complete Helm Values File

Save this as `loki-stack-values.yaml`:

```yaml
# Production-ready Grafana Loki Stack Configuration for OpenShift
# Optimized for SonataFlow workflow log aggregation

# ==============================================================================
# Loki Configuration
# ==============================================================================
loki:
  enabled: true

  # Use persistent storage for production
  persistence:
    enabled: true
    storageClassName: gp3-csi  # Adjust based on your OpenShift storage class
    accessModes:
      - ReadWriteOnce
    size: 50Gi

  # OpenShift-specific security context
  securityContext:
    fsGroup: 10001
    runAsGroup: 10001
    runAsNonRoot: true
    runAsUser: 10001

  containerSecurityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop:
        - ALL
    readOnlyRootFilesystem: true
    runAsNonRoot: true
    runAsUser: 10001

  # Resource limits for production
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi

  # Loki configuration
  config:
    # Authentication disabled for internal cluster use
    auth_enabled: false

    # Server configuration
    server:
      http_listen_port: 3100
      grpc_listen_port: 9096
      log_level: info

    # Ingester configuration for log streams
    ingester:
      chunk_idle_period: 3m
      chunk_block_size: 262144
      chunk_retain_period: 1m
      max_transfer_retries: 0
      wal:
        enabled: true
        dir: /data/loki/wal
      lifecycler:
        ring:
          kvstore:
            store: inmemory
          replication_factor: 1

    # Limits configuration
    limits_config:
      enforce_metric_name: false
      reject_old_samples: true
      reject_old_samples_max_age: 168h  # 7 days
      ingestion_rate_mb: 16
      ingestion_burst_size_mb: 32
      max_query_length: 721h  # 30 days
      max_query_parallelism: 16
      max_streams_per_user: 10000
      max_global_streams_per_user: 10000
      max_entries_limit_per_query: 10000
      max_cache_freshness_per_query: 10m

    # Schema configuration for storage
    schema_config:
      configs:
        - from: 2024-01-01
          store: boltdb-shipper
          object_store: filesystem
          schema: v11
          index:
            prefix: index_
            period: 24h

    # Storage configuration
    storage_config:
      boltdb_shipper:
        active_index_directory: /data/loki/boltdb-shipper-active
        cache_location: /data/loki/boltdb-shipper-cache
        cache_ttl: 24h
        shared_store: filesystem
      filesystem:
        directory: /data/loki/chunks

    # Chunk store configuration
    chunk_store_config:
      max_look_back_period: 720h  # 30 days

    # Table manager configuration
    table_manager:
      retention_deletes_enabled: true
      retention_period: 720h  # 30 days

    # Query configuration
    query_range:
      align_queries_with_step: true
      max_retries: 5
      cache_results: true
      results_cache:
        cache:
          enable_fifocache: true
          fifocache:
            max_size_bytes: 500MB
            validity: 24h

    # Compactor for log retention
    compactor:
      working_directory: /data/loki/compactor
      shared_store: filesystem
      compaction_interval: 10m
      retention_enabled: true
      retention_delete_delay: 2h
      retention_delete_worker_count: 150

  # Service configuration
  service:
    type: ClusterIP
    port: 3100
    targetPort: 3100
    annotations: {}

  # Enable ServiceMonitor for Prometheus monitoring (optional)
  serviceMonitor:
    enabled: false

  # Node selector for scheduling
  nodeSelector: {}

  # Tolerations
  tolerations: []

  # Affinity rules
  affinity: {}

# ==============================================================================
# Promtail Configuration
# ==============================================================================
promtail:
  enabled: true

  # OpenShift-specific security context
  securityContext:
    runAsUser: 0  # Required for accessing /var/log/pods
    runAsGroup: 0
    fsGroup: 0
    privileged: false
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false

  # Resource limits
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

  # Run as DaemonSet to collect logs from all nodes
  daemonset:
    enabled: true

  # Promtail configuration
  config:
    # Server configuration
    server:
      http_listen_port: 3101
      grpc_listen_port: 0
      log_level: info

    # Loki client configuration
    clients:
      - url: http://loki:3100/loki/api/v1/push
        tenant_id: ""
        batchwait: 1s
        batchsize: 1048576
        backoff_config:
          min_period: 500ms
          max_period: 5m
          max_retries: 10
        timeout: 10s

    # Position tracking
    positions:
      filename: /run/promtail/positions.yaml

    # Target configuration for SonataFlow pods
    target_config:
      sync_period: 10s

    # Scrape configurations
    scrape_configs:
      # ==================================================================
      # SonataFlow Workflow Pods - JSON Logs
      # ==================================================================
      - job_name: sonataflow-workflows

        # Kubernetes service discovery for pods
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - sonataflow-infra
                - default
                # Add other namespaces where workflows are deployed

        # Pipeline stages for processing logs
        pipeline_stages:
          # Stage 1: Extract Kubernetes metadata
          - docker: {}

          # Stage 2: Parse JSON logs
          - json:
              expressions:
                timestamp: timestamp
                level: level
                logger: logger
                message: message
                thread: thread
                exception: exception
                # MDC fields
                processInstanceId: MDC.processInstanceId
                traceId: MDC.traceId
                spanId: MDC.spanId
                workflowId: MDC.workflowId
                state: MDC.state
                userId: MDC.userId
                businessKey: MDC.businessKey

          # Stage 3: Parse timestamp
          - timestamp:
              source: timestamp
              format: RFC3339Nano
              fallback_formats:
                - "2006-01-02T15:04:05.999Z"
                - "2006-01-02T15:04:05Z07:00"

          # Stage 4: Extract log level
          - labels:
              level:
              logger:

          # Stage 5: Add MDC fields as labels for filtering
          - labels:
              processInstanceId:
              traceId:
              spanId:
              workflowId:
              state:

          # Stage 6: Add structured metadata
          - structured_metadata:
              thread:
              userId:
              businessKey:

          # Stage 7: Output formatting
          - output:
              source: message

        # Relabeling rules for Kubernetes metadata
        relabel_configs:
          # Only scrape pods with SonataFlow labels
          - source_labels: [__meta_kubernetes_pod_label_sonataflow_org_workflow_app]
            action: keep
            regex: (.+)

          # Add namespace label
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace

          # Add pod name label
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod

          # Add container name label
          - source_labels: [__meta_kubernetes_pod_container_name]
            target_label: container

          # Add workflow name from pod label
          - source_labels: [__meta_kubernetes_pod_label_sonataflow_org_workflow_app]
            target_label: workflow_name

          # Add workflow version from annotation
          - source_labels: [__meta_kubernetes_pod_annotation_sonataflow_org_version]
            target_label: workflow_version

          # Add node name
          - source_labels: [__meta_kubernetes_pod_node_name]
            target_label: node

          # Add pod IP
          - source_labels: [__meta_kubernetes_pod_ip]
            target_label: pod_ip

          # Add custom labels from pod annotations
          - source_labels: [__meta_kubernetes_pod_annotation_app]
            target_label: app

          # Add environment label if present
          - source_labels: [__meta_kubernetes_pod_label_environment]
            target_label: environment

          # Set the log path
          - source_labels: [__meta_kubernetes_pod_uid, __meta_kubernetes_pod_container_name]
            target_label: __path__
            separator: /
            replacement: /var/log/pods/*$1/*/*.log

      # ==================================================================
      # SonataFlow Platform Services (Data Index, Job Service)
      # ==================================================================
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
                serviceType: MDC.serviceType
          - timestamp:
              source: timestamp
              format: RFC3339Nano
          - labels:
              level:
              serviceType:
          - output:
              source: message

        relabel_configs:
          # Match Data Index and Job Service pods
          - source_labels: [__meta_kubernetes_pod_label_sonataflow_org_service]
            action: keep
            regex: (sonataflow-platform-data-index-service|sonataflow-platform-jobs-service)

          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace

          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod

          - source_labels: [__meta_kubernetes_pod_label_sonataflow_org_service]
            target_label: service_name

          - source_labels: [__meta_kubernetes_pod_uid, __meta_kubernetes_pod_container_name]
            target_label: __path__
            separator: /
            replacement: /var/log/pods/*$1/*/*.log

  # Volume mounts for accessing logs
  volumeMounts:
    - name: run
      mountPath: /run/promtail
    - name: pods
      mountPath: /var/log/pods
      readOnly: true
    - name: docker
      mountPath: /var/lib/docker/containers
      readOnly: true

  volumes:
    - name: run
      emptyDir: {}
    - name: pods
      hostPath:
        path: /var/log/pods
    - name: docker
      hostPath:
        path: /var/lib/docker/containers

  # Service account with proper permissions
  serviceAccount:
    create: true
    name: promtail

  # RBAC permissions
  rbac:
    create: true
    pspEnabled: false

# ==============================================================================
# Grafana Configuration
# ==============================================================================
grafana:
  enabled: true

  # Admin credentials (change these!)
  adminUser: admin
  adminPassword: admin123  # CHANGE THIS IN PRODUCTION!

  # OpenShift-specific security context
  securityContext:
    runAsUser: 472
    runAsGroup: 472
    fsGroup: 472
    runAsNonRoot: true

  containerSecurityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop:
        - ALL
    readOnlyRootFilesystem: false

  # Resource limits
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 2Gi

  # Persistence for dashboards and settings
  persistence:
    enabled: true
    storageClassName: gp3-csi
    accessModes:
      - ReadWriteOnce
    size: 10Gi

  # Grafana configuration
  grafana.ini:
    server:
      root_url: "%(protocol)s://%(domain)s:%(http_port)s/"
      serve_from_sub_path: false

    security:
      admin_user: admin
      admin_password: admin123  # CHANGE THIS IN PRODUCTION!
      disable_gravatar: true

    auth:
      disable_login_form: false
      disable_signout_menu: false

    auth.anonymous:
      enabled: false

    analytics:
      check_for_updates: false
      reporting_enabled: false

    log:
      mode: console
      level: info

    alerting:
      enabled: true

  # Datasources configuration
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Loki
          type: loki
          access: proxy
          url: http://loki:3100
          isDefault: true
          jsonData:
            maxLines: 1000
            derivedFields:
              # Enable trace correlation
              - datasourceUid: tempo  # If you have Tempo for tracing
                matcherRegex: "traceId=(\\w+)"
                name: TraceID
                url: "$${__value.raw}"
              # Link to process instance
              - matcherRegex: "processInstanceId=(\\w+)"
                name: ProcessInstance
                url: "/workflows/instances/$${__value.raw}"
          editable: true

  # Dashboard providers
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: 'sonataflow'
          orgId: 1
          folder: 'SonataFlow Workflows'
          type: file
          disableDeletion: false
          editable: true
          options:
            path: /var/lib/grafana/dashboards/sonataflow

  # Pre-load dashboards
  dashboardsConfigMaps:
    sonataflow: "grafana-sonataflow-dashboards"

  # Service configuration
  service:
    type: ClusterIP
    port: 80
    targetPort: 3000

  # Create OpenShift Route for external access
  route:
    enabled: true
    host: ""  # OpenShift will auto-generate
    tls:
      enabled: true
      termination: edge
      insecureEdgeTerminationPolicy: Redirect

  # Plugins to install
  plugins:
    - grafana-piechart-panel
    - grafana-worldmap-panel

  # Environment variables
  env:
    GF_INSTALL_PLUGINS: "grafana-piechart-panel,grafana-worldmap-panel"

# ==============================================================================
# Additional Components
# ==============================================================================

# Fluent Bit (disabled, using Promtail instead)
fluent-bit:
  enabled: false

# Log Gateway (disabled for simplicity)
logstash:
  enabled: false

# File Beat (disabled, using Promtail instead)
filebeat:
  enabled: false
```

## Post-Installation Steps

### 1. Verify Installation

```bash
# Check all pods are running
oc get pods -n sonataflow-observability

# Expected output:
# NAME                            READY   STATUS    RESTARTS   AGE
# loki-0                          1/1     Running   0          5m
# promtail-xxxxx                  1/1     Running   0          5m
# grafana-xxxxxxxxxx-xxxxx        1/1     Running   0          5m
```

### 2. Access Grafana

```bash
# Get the Grafana route
oc get route grafana -n sonataflow-observability

# Get admin password (if you want to retrieve it)
oc get secret loki-stack-grafana -n sonataflow-observability \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### 3. Verify Loki is Receiving Logs

```bash
# Port-forward to Loki
oc port-forward -n sonataflow-observability svc/loki 3100:3100

# Query Loki (in another terminal)
curl -G -s "http://localhost:3100/loki/api/v1/query" \
  --data-urlencode 'query={job="sonataflow-workflows"}' | jq
```

### 4. Create ConfigMap for Dashboard

See the [Grafana Dashboard](grafana-dashboard.md) page for the complete dashboard JSON.

```bash
# Create ConfigMap with dashboard
oc create configmap grafana-sonataflow-dashboards \
  -n sonataflow-observability \
  --from-file=sonataflow-dashboard.json

# Label it for auto-discovery
oc label configmap grafana-sonataflow-dashboards \
  -n sonataflow-observability \
  grafana_dashboard=1
```

## Configuration Options

### Storage Classes

Adjust `storageClassName` based on your OpenShift cluster:

- AWS: `gp3-csi` or `gp2`
- Azure: `managed-premium`
- GCP: `standard-rwo`
- On-premise: Check with `oc get storageclass`

### Namespace Configuration

To monitor workflows in multiple namespaces, update the `promtail.config.scrape_configs.kubernetes_sd_configs.namespaces.names` section:

```yaml
namespaces:
  names:
    - sonataflow-infra
    - default
    - my-workflow-namespace
    - another-namespace
```

### Resource Tuning

Adjust resources based on your cluster size and log volume:

- **Small cluster** (< 10 workflows): Use default values
- **Medium cluster** (10-50 workflows): Double memory limits
- **Large cluster** (50+ workflows): Use 4Gi+ for Loki, consider distributed deployment

### Retention Configuration

Modify log retention in the Loki configuration:

```yaml
limits_config:
  reject_old_samples_max_age: 168h  # 7 days

table_manager:
  retention_period: 720h  # 30 days
```

## Troubleshooting

### Promtail Not Discovering Pods

Check the Promtail logs:

```bash
oc logs -n sonataflow-observability -l app.kubernetes.io/name=promtail --tail=100
```

Verify pod labels match:

```bash
oc get pods -n sonataflow-infra --show-labels | grep sonataflow
```

### Loki Storage Issues

Check PVC status:

```bash
oc get pvc -n sonataflow-observability
```

Increase storage if needed:

```bash
oc patch pvc loki -n sonataflow-observability \
  -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```

### Grafana Route Not Working

Create route manually:

```bash
oc create route edge grafana \
  --service=grafana \
  -n sonataflow-observability
```

## Security Hardening

### 1. Change Default Passwords

Create a secret for Grafana admin password:

```bash
oc create secret generic grafana-admin \
  -n sonataflow-observability \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=$(openssl rand -base64 32)
```

Update values:

```yaml
grafana:
  admin:
    existingSecret: grafana-admin
    userKey: admin-user
    passwordKey: admin-password
```

### 2. Enable TLS for Internal Communication

```yaml
loki:
  config:
    server:
      http_tls_config:
        cert_file: /etc/loki/tls/tls.crt
        key_file: /etc/loki/tls/tls.key
```

### 3. Integrate with OpenShift OAuth

```yaml
grafana:
  grafana.ini:
    auth.proxy:
      enabled: true
      header_name: X-Forwarded-User
      header_property: username
      auto_sign_up: true
```

## Next Steps

- [Configure Promtail](promtail-config.md) for advanced log processing
- [Import Grafana Dashboard](grafana-dashboard.md) for visualization
- [Learn LogQL Queries](logql-queries.md) for log analysis
- [Deploy RBAC Manifests](deployment-manifests.md) for security

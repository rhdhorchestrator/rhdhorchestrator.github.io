---
title: Deployment Manifests
date: "2025-11-24"
weight: 5
---

# Complete Deployment Manifests for PLG Stack

This page provides production-ready Kubernetes/OpenShift YAML manifests for deploying the PLG stack with proper security, RBAC, and configurations.

## Quick Deployment

```bash
# Clone or download all manifests
# Apply in order:
oc apply -f 01-namespace.yaml
oc apply -f 02-rbac.yaml
oc apply -f 03-loki-config.yaml
oc apply -f 04-loki-deployment.yaml
oc apply -f 05-promtail-config.yaml
oc apply -f 06-promtail-daemonset.yaml
oc apply -f 07-grafana-config.yaml
oc apply -f 08-grafana-deployment.yaml
oc apply -f 09-routes.yaml
```

## Complete Manifests

### 1. Namespace

File: `01-namespace.yaml`

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: sonataflow-observability
  labels:
    name: sonataflow-observability
    monitoring: enabled
    openshift.io/cluster-monitoring: "true"
  annotations:
    openshift.io/description: "Observability stack for SonataFlow workflows using PLG (Promtail, Loki, Grafana)"
    openshift.io/display-name: "SonataFlow Observability"

---
# Resource Quota to prevent resource exhaustion
apiVersion: v1
kind: ResourceQuota
metadata:
  name: observability-quota
  namespace: sonataflow-observability
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "10"
    requests.storage: 200Gi

---
# Limit Range for default resource constraints
apiVersion: v1
kind: LimitRange
metadata:
  name: observability-limits
  namespace: sonataflow-observability
spec:
  limits:
    - max:
        cpu: "4"
        memory: 8Gi
      min:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 250m
        memory: 256Mi
      type: Container
    - max:
        storage: 100Gi
      min:
        storage: 1Gi
      type: PersistentVolumeClaim

---
# Network Policy to control traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-observability-traffic
  namespace: sonataflow-observability
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow from same namespace
    - from:
        - podSelector: {}
    # Allow from SonataFlow namespaces (Promtail scraping)
    - from:
        - namespaceSelector:
            matchLabels:
              monitoring: enabled
    # Allow from OpenShift router (for Routes)
    - from:
        - namespaceSelector:
            matchLabels:
              policy-group.network.openshift.io/ingress: ""
  egress:
    # Allow to same namespace
    - to:
        - podSelector: {}
    # Allow to SonataFlow namespaces (for log collection)
    - to:
        - namespaceSelector: {}
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: openshift-dns
      ports:
        - protocol: UDP
          port: 53
    # Allow to API server
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: default
      ports:
        - protocol: TCP
          port: 443
```

### 2. RBAC (Service Accounts, Roles, ClusterRoles)

File: `02-rbac.yaml`

```yaml
---
# Loki Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: loki
  namespace: sonataflow-observability
  labels:
    app: loki

---
# Promtail Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: promtail
  namespace: sonataflow-observability
  labels:
    app: promtail

---
# Grafana Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: grafana
  namespace: sonataflow-observability
  labels:
    app: grafana

---
# ClusterRole for Promtail (needs to read pods across namespaces)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: promtail
  labels:
    app: promtail
rules:
  # Read pods and namespaces for service discovery
  - apiGroups: [""]
    resources:
      - pods
      - pods/log
      - nodes
      - services
      - endpoints
    verbs:
      - get
      - list
      - watch

  # Read namespaces for filtering
  - apiGroups: [""]
    resources:
      - namespaces
    verbs:
      - get
      - list
      - watch

  # Access configmaps for position tracking
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
      - list
      - watch

---
# ClusterRoleBinding for Promtail
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: promtail
  labels:
    app: promtail
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: promtail
subjects:
  - kind: ServiceAccount
    name: promtail
    namespace: sonataflow-observability

---
# Role for Loki (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: loki
  namespace: sonataflow-observability
  labels:
    app: loki
rules:
  # Access to ConfigMaps for configuration
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
      - list
      - watch

  # Access to Secrets
  - apiGroups: [""]
    resources:
      - secrets
    verbs:
      - get
      - list
      - watch

---
# RoleBinding for Loki
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: loki
  namespace: sonataflow-observability
  labels:
    app: loki
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: loki
subjects:
  - kind: ServiceAccount
    name: loki
    namespace: sonataflow-observability

---
# Role for Grafana
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: grafana
  namespace: sonataflow-observability
  labels:
    app: grafana
rules:
  # Access to ConfigMaps for dashboards
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
      - list
      - watch

  # Access to Secrets for credentials
  - apiGroups: [""]
    resources:
      - secrets
    verbs:
      - get
      - list
      - watch

---
# RoleBinding for Grafana
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: grafana
  namespace: sonataflow-observability
  labels:
    app: grafana
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: grafana
subjects:
  - kind: ServiceAccount
    name: grafana
    namespace: sonataflow-observability

---
# SecurityContextConstraints for Promtail (OpenShift-specific)
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: promtail-scc
  labels:
    app: promtail
allowHostDirVolumePlugin: true
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: false
allowPrivilegedContainer: false
allowedCapabilities: null
defaultAddCapabilities: null
fsGroup:
  type: MustRunAs
  ranges:
    - min: 1
      max: 65535
readOnlyRootFilesystem: true
requiredDropCapabilities:
  - ALL
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - hostPath
  - persistentVolumeClaim
  - projected
  - secret
users:
  - system:serviceaccount:sonataflow-observability:promtail
```

### 3. Loki Configuration

File: `03-loki-config.yaml`

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-config
  namespace: sonataflow-observability
  labels:
    app: loki
data:
  loki.yaml: |
    auth_enabled: false

    server:
      http_listen_port: 3100
      grpc_listen_port: 9096
      log_level: info

    common:
      path_prefix: /data/loki
      storage:
        filesystem:
          chunks_directory: /data/loki/chunks
          rules_directory: /data/loki/rules
      replication_factor: 1
      ring:
        kvstore:
          store: inmemory

    schema_config:
      configs:
        - from: 2024-01-01
          store: boltdb-shipper
          object_store: filesystem
          schema: v11
          index:
            prefix: index_
            period: 24h

    storage_config:
      boltdb_shipper:
        active_index_directory: /data/loki/boltdb-shipper-active
        cache_location: /data/loki/boltdb-shipper-cache
        cache_ttl: 24h
        shared_store: filesystem
      filesystem:
        directory: /data/loki/chunks

    compactor:
      working_directory: /data/loki/compactor
      shared_store: filesystem
      compaction_interval: 10m
      retention_enabled: true
      retention_delete_delay: 2h
      retention_delete_worker_count: 150

    limits_config:
      enforce_metric_name: false
      reject_old_samples: true
      reject_old_samples_max_age: 168h
      ingestion_rate_mb: 16
      ingestion_burst_size_mb: 32
      max_query_length: 721h
      max_query_parallelism: 16
      max_streams_per_user: 10000
      max_global_streams_per_user: 10000
      max_entries_limit_per_query: 10000

    chunk_store_config:
      max_look_back_period: 720h

    table_manager:
      retention_deletes_enabled: true
      retention_period: 720h

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

    ruler:
      storage:
        type: local
        local:
          directory: /data/loki/rules
      rule_path: /data/loki/rules-temp
      alertmanager_url: http://alertmanager:9093
      ring:
        kvstore:
          store: inmemory
      enable_api: true
```

### 4. Loki Deployment

File: `04-loki-deployment.yaml`

```yaml
---
# Persistent Volume Claim for Loki
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: loki-data
  namespace: sonataflow-observability
  labels:
    app: loki
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: gp3-csi  # Adjust based on your cluster

---
# Loki StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: loki
  namespace: sonataflow-observability
  labels:
    app: loki
spec:
  serviceName: loki
  replicas: 1
  selector:
    matchLabels:
      app: loki
  template:
    metadata:
      labels:
        app: loki
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3100"
    spec:
      serviceAccountName: loki

      securityContext:
        fsGroup: 10001
        runAsGroup: 10001
        runAsNonRoot: true
        runAsUser: 10001

      containers:
        - name: loki
          image: grafana/loki:2.9.3
          imagePullPolicy: IfNotPresent

          args:
            - -config.file=/etc/loki/loki.yaml

          ports:
            - name: http-metrics
              containerPort: 3100
              protocol: TCP
            - name: grpc
              containerPort: 9096
              protocol: TCP

          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 10001

          livenessProbe:
            httpGet:
              path: /ready
              port: http-metrics
            initialDelaySeconds: 45
            periodSeconds: 10
            timeoutSeconds: 1
            successThreshold: 1
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /ready
              port: http-metrics
            initialDelaySeconds: 45
            periodSeconds: 10
            timeoutSeconds: 1
            successThreshold: 1
            failureThreshold: 3

          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 2000m
              memory: 4Gi

          volumeMounts:
            - name: config
              mountPath: /etc/loki
            - name: data
              mountPath: /data
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: config
          configMap:
            name: loki-config
        - name: data
          persistentVolumeClaim:
            claimName: loki-data
        - name: tmp
          emptyDir: {}

---
# Loki Service
apiVersion: v1
kind: Service
metadata:
  name: loki
  namespace: sonataflow-observability
  labels:
    app: loki
spec:
  type: ClusterIP
  ports:
    - name: http-metrics
      port: 3100
      targetPort: http-metrics
      protocol: TCP
    - name: grpc
      port: 9096
      targetPort: grpc
      protocol: TCP
  selector:
    app: loki

---
# Headless Service for StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: loki-headless
  namespace: sonataflow-observability
  labels:
    app: loki
spec:
  clusterIP: None
  ports:
    - name: http-metrics
      port: 3100
      targetPort: http-metrics
      protocol: TCP
  selector:
    app: loki
```

### 5. Promtail Configuration

File: `05-promtail-config.yaml`

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: sonataflow-observability
  labels:
    app: promtail
data:
  promtail.yaml: |
    server:
      http_listen_port: 3101
      grpc_listen_port: 0
      log_level: info

    positions:
      filename: /run/promtail/positions.yaml

    clients:
      - url: http://loki.sonataflow-observability.svc.cluster.local:3100/loki/api/v1/push
        tenant_id: ""
        batchwait: 1s
        batchsize: 1048576
        backoff_config:
          min_period: 500ms
          max_period: 5m
          max_retries: 10
        timeout: 10s

    target_config:
      sync_period: 10s

    scrape_configs:
      # SonataFlow Workflow Pods
      - job_name: sonataflow-workflows
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - sonataflow-infra
                - default

        pipeline_stages:
          - docker: {}
          - json:
              expressions:
                timestamp: timestamp
                level: level
                logger: logger
                message: message
                processInstanceId: MDC.processInstanceId
                traceId: MDC.traceId
                spanId: MDC.spanId
                workflowId: MDC.workflowId
                state: MDC.state
          - timestamp:
              source: timestamp
              format: RFC3339Nano
          - labels:
              level:
              processInstanceId:
              traceId:
              workflowId:
              state:
          - output:
              source: message

        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_sonataflow_org_workflow_app]
            action: keep
            regex: (.+)
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
          - source_labels: [__meta_kubernetes_pod_container_name]
            target_label: container
          - source_labels: [__meta_kubernetes_pod_label_sonataflow_org_workflow_app]
            target_label: workflow_name
          - source_labels: [__meta_kubernetes_pod_node_name]
            target_label: node
          - source_labels: [__meta_kubernetes_pod_uid, __meta_kubernetes_pod_container_name]
            target_label: __path__
            separator: /
            replacement: /var/log/pods/*$1/*/*.log
```

### 6. Promtail DaemonSet

File: `06-promtail-daemonset.yaml`

```yaml
---
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

      securityContext:
        runAsUser: 0
        runAsGroup: 0
        fsGroup: 0

      hostNetwork: false
      dnsPolicy: ClusterFirst

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

          env:
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName

          ports:
            - name: http-metrics
              containerPort: 3101
              protocol: TCP

          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: false
            runAsUser: 0

          livenessProbe:
            httpGet:
              path: /ready
              port: http-metrics
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 1
            successThreshold: 1
            failureThreshold: 5

          readinessProbe:
            httpGet:
              path: /ready
              port: http-metrics
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 1
            successThreshold: 1
            failureThreshold: 5

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi

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

### 7. Grafana Configuration

File: `07-grafana-config.yaml`

```yaml
---
# Grafana Admin Credentials Secret
apiVersion: v1
kind: Secret
metadata:
  name: grafana-admin
  namespace: sonataflow-observability
  labels:
    app: grafana
type: Opaque
stringData:
  admin-user: admin
  admin-password: changeme123  # CHANGE THIS!

---
# Grafana Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-config
  namespace: sonataflow-observability
  labels:
    app: grafana
data:
  grafana.ini: |
    [server]
    root_url = %(protocol)s://%(domain)s:%(http_port)s/
    serve_from_sub_path = false

    [security]
    admin_user = admin
    disable_gravatar = true

    [auth]
    disable_login_form = false
    disable_signout_menu = false

    [auth.anonymous]
    enabled = false

    [analytics]
    check_for_updates = false
    reporting_enabled = false

    [log]
    mode = console
    level = info

    [alerting]
    enabled = true

---
# Grafana Datasources
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: sonataflow-observability
  labels:
    app: grafana
    grafana_datasource: "1"
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - name: Loki
        type: loki
        access: proxy
        url: http://loki.sonataflow-observability.svc.cluster.local:3100
        isDefault: true
        jsonData:
          maxLines: 1000
        editable: true
```

### 8. Grafana Deployment

File: `08-grafana-deployment.yaml`

```yaml
---
# PVC for Grafana
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-data
  namespace: sonataflow-observability
  labels:
    app: grafana
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp3-csi

---
# Grafana Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: sonataflow-observability
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      serviceAccountName: grafana

      securityContext:
        runAsUser: 472
        runAsGroup: 472
        fsGroup: 472
        runAsNonRoot: true

      containers:
        - name: grafana
          image: grafana/grafana:10.2.2
          imagePullPolicy: IfNotPresent

          ports:
            - name: http
              containerPort: 3000
              protocol: TCP

          env:
            - name: GF_SECURITY_ADMIN_USER
              valueFrom:
                secretKeyRef:
                  name: grafana-admin
                  key: admin-user
            - name: GF_SECURITY_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-admin
                  key: admin-password
            - name: GF_INSTALL_PLUGINS
              value: "grafana-piechart-panel"

          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: false

          livenessProbe:
            httpGet:
              path: /api/health
              port: http
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 30
            successThreshold: 1
            failureThreshold: 10

          readinessProbe:
            httpGet:
              path: /api/health
              port: http
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 30
            successThreshold: 1
            failureThreshold: 3

          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 2Gi

          volumeMounts:
            - name: config
              mountPath: /etc/grafana
            - name: datasources
              mountPath: /etc/grafana/provisioning/datasources
            - name: data
              mountPath: /var/lib/grafana

      volumes:
        - name: config
          configMap:
            name: grafana-config
        - name: datasources
          configMap:
            name: grafana-datasources
        - name: data
          persistentVolumeClaim:
            claimName: grafana-data

---
# Grafana Service
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: sonataflow-observability
  labels:
    app: grafana
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
  selector:
    app: grafana
```

### 9. OpenShift Routes

File: `09-routes.yaml`

```yaml
---
# Grafana Route
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: grafana
  namespace: sonataflow-observability
  labels:
    app: grafana
  annotations:
    haproxy.router.openshift.io/timeout: 4m
    haproxy.router.openshift.io/disable_cookies: "true"
spec:
  to:
    kind: Service
    name: grafana
    weight: 100
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: None

---
# Loki Route (Optional - for external access)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: loki
  namespace: sonataflow-observability
  labels:
    app: loki
  annotations:
    haproxy.router.openshift.io/timeout: 4m
spec:
  to:
    kind: Service
    name: loki
    weight: 100
  port:
    targetPort: http-metrics
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: None
```

## Additional Configuration Files

### 10. Monitoring (ServiceMonitors for Prometheus)

File: `10-monitoring.yaml`

```yaml
---
# ServiceMonitor for Loki (if using Prometheus Operator)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: loki
  namespace: sonataflow-observability
  labels:
    app: loki
spec:
  selector:
    matchLabels:
      app: loki
  endpoints:
    - port: http-metrics
      interval: 30s
      path: /metrics

---
# ServiceMonitor for Promtail
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: promtail
  namespace: sonataflow-observability
  labels:
    app: promtail
spec:
  selector:
    matchLabels:
      app: promtail
  endpoints:
    - port: http-metrics
      interval: 30s
      path: /metrics

---
# ServiceMonitor for Grafana
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: grafana
  namespace: sonataflow-observability
  labels:
    app: grafana
spec:
  selector:
    matchLabels:
      app: grafana
  endpoints:
    - port: http
      interval: 30s
      path: /metrics
```

## Deployment Guide

### Step 1: Pre-deployment Checks

```bash
# Verify you have cluster-admin privileges
oc whoami
oc auth can-i create namespace

# Check available storage classes
oc get storageclass

# Verify SonataFlow namespaces exist
oc get namespace sonataflow-infra
```

### Step 2: Update Configuration

Before deploying, update these values:

1. **Storage Class**: In all PVC manifests, change `storageClassName`:
   ```yaml
   storageClassName: gp3-csi  # Change to your storage class
   ```

2. **Grafana Password**: In `07-grafana-config.yaml`:
   ```yaml
   admin-password: changeme123  # Use a strong password
   ```

3. **Namespaces**: In `05-promtail-config.yaml`, add your workflow namespaces:
   ```yaml
   namespaces:
     names:
       - sonataflow-infra
       - default
       - your-namespace  # Add yours here
   ```

### Step 3: Deploy

```bash
# Deploy in order
oc apply -f 01-namespace.yaml
oc apply -f 02-rbac.yaml
oc apply -f 03-loki-config.yaml
oc apply -f 04-loki-deployment.yaml
oc apply -f 05-promtail-config.yaml
oc apply -f 06-promtail-daemonset.yaml
oc apply -f 07-grafana-config.yaml
oc apply -f 08-grafana-deployment.yaml
oc apply -f 09-routes.yaml

# Optional: monitoring
oc apply -f 10-monitoring.yaml
```

### Step 4: Verify Deployment

```bash
# Check all pods are running
oc get pods -n sonataflow-observability

# Expected output:
# NAME                       READY   STATUS    RESTARTS   AGE
# loki-0                     1/1     Running   0          2m
# promtail-xxxxx             1/1     Running   0          2m
# promtail-yyyyy             1/1     Running   0          2m
# grafana-xxxxxxxxxx-zzzzz   1/1     Running   0          2m

# Check PVCs are bound
oc get pvc -n sonataflow-observability

# Get Grafana URL
oc get route grafana -n sonataflow-observability -o jsonpath='{.spec.host}'
```

### Step 5: Access Grafana

```bash
# Get the Grafana URL
GRAFANA_URL=$(oc get route grafana -n sonataflow-observability -o jsonpath='{.spec.host}')
echo "Grafana URL: https://${GRAFANA_URL}"

# Get admin password (if you forgot)
oc get secret grafana-admin -n sonataflow-observability -o jsonpath='{.data.admin-password}' | base64 -d
echo
```

### Step 6: Import Dashboard

```bash
# Create dashboard ConfigMap (see grafana-dashboard.md for JSON)
oc create configmap grafana-sonataflow-dashboards \
  -n sonataflow-observability \
  --from-file=sonataflow-dashboard.json

# Label for auto-discovery
oc label configmap grafana-sonataflow-dashboards \
  -n sonataflow-observability \
  grafana_dashboard=1

# Restart Grafana
oc rollout restart deployment/grafana -n sonataflow-observability
```

## Troubleshooting

### Pods Not Starting

Check events:
```bash
oc get events -n sonataflow-observability --sort-by='.lastTimestamp'
```

Check pod logs:
```bash
oc logs -n sonataflow-observability <pod-name>
```

### PVC Not Binding

Check storage class:
```bash
oc get storageclass
oc describe pvc -n sonataflow-observability
```

### Promtail Not Scraping Logs

Check ServiceAccount permissions:
```bash
oc auth can-i list pods --as=system:serviceaccount:sonataflow-observability:promtail -n sonataflow-infra
```

Apply SCC:
```bash
oc adm policy add-scc-to-user promtail-scc -z promtail -n sonataflow-observability
```

### Grafana Can't Connect to Loki

Test connectivity:
```bash
oc exec -it -n sonataflow-observability deployment/grafana -- \
  wget -O- http://loki.sonataflow-observability.svc.cluster.local:3100/ready
```

## Cleanup

To remove the entire stack:

```bash
# Delete all resources
oc delete -f 10-monitoring.yaml
oc delete -f 09-routes.yaml
oc delete -f 08-grafana-deployment.yaml
oc delete -f 07-grafana-config.yaml
oc delete -f 06-promtail-daemonset.yaml
oc delete -f 05-promtail-config.yaml
oc delete -f 04-loki-deployment.yaml
oc delete -f 03-loki-config.yaml
oc delete -f 02-rbac.yaml

# Delete PVCs (this will delete all data!)
oc delete pvc -n sonataflow-observability --all

# Delete namespace
oc delete -f 01-namespace.yaml

# Remove ClusterRole and ClusterRoleBinding
oc delete clusterrole promtail
oc delete clusterrolebinding promtail
oc delete scc promtail-scc
```

## Security Hardening

### 1. Use Secrets for Sensitive Data

Instead of ConfigMaps for credentials:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: loki-credentials
  namespace: sonataflow-observability
type: Opaque
stringData:
  username: admin
  password: $(openssl rand -base64 32)
```

### 2. Enable TLS Between Components

Create certificates and update service URLs:

```yaml
clients:
  - url: https://loki.sonataflow-observability.svc.cluster.local:3100/loki/api/v1/push
    tls_config:
      ca_file: /etc/promtail/ca.crt
      cert_file: /etc/promtail/tls.crt
      key_file: /etc/promtail/tls.key
```

### 3. Integrate with OpenShift OAuth

Update Grafana configuration:

```yaml
[auth.proxy]
enabled = true
header_name = X-Forwarded-User
header_property = username
auto_sign_up = true
```

## Next Steps

- [Configure LogQL Queries](logql-queries.md) for log analysis
- [Import Grafana Dashboard](grafana-dashboard.md) for visualization
- [Review Helm Values](helm-values.md) for alternative deployment
- [Optimize Promtail](promtail-config.md) for your use case

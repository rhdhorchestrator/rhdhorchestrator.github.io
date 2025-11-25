---
title: Quick Start Guide
date: "2025-11-24"
weight: 0
---

# PLG Stack Quick Start for SonataFlow

Get the PLG (Promtail + Loki + Grafana) stack running on OpenShift in under 15 minutes.

## Prerequisites

- OpenShift 4.12+
- `oc` CLI installed and logged in
- Cluster admin privileges
- SonataFlow Operator installed
- Helm 3.x (for Helm installation method)

## Quick Installation (Helm)

The fastest way to get started:

### Step 1: Add Helm Repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### Step 2: Create Namespace

```bash
oc create namespace sonataflow-observability
```

### Step 3: Deploy Stack

Download the [complete Helm values file](helm-values.md#complete-helm-values-file) and save as `loki-stack-values.yaml`, then:

```bash
# Install the Loki stack
helm install loki-stack grafana/loki-stack \
  --namespace sonataflow-observability \
  --values loki-stack-values.yaml \
  --wait
```

### Step 4: Create OpenShift Route for Grafana

```bash
# Create route
oc create route edge grafana \
  --service=loki-stack-grafana \
  --namespace=sonataflow-observability

# Get the URL
oc get route grafana -n sonataflow-observability -o jsonpath='{.spec.host}'
```

### Step 5: Access Grafana

```bash
# Get admin password
export GRAFANA_PASSWORD=$(oc get secret loki-stack-grafana \
  -n sonataflow-observability \
  -o jsonpath="{.data.admin-password}" | base64 --decode)

echo "Username: admin"
echo "Password: ${GRAFANA_PASSWORD}"

# Get URL
export GRAFANA_URL=$(oc get route grafana -n sonataflow-observability -o jsonpath='{.spec.host}')
echo "Grafana URL: https://${GRAFANA_URL}"
```

### Step 6: Import Dashboard

1. Download the [dashboard JSON](grafana-dashboard.md#complete-dashboard-json)
2. Open Grafana at the URL from Step 5
3. Login with admin credentials
4. Navigate to **Dashboards** → **Import**
5. Paste the JSON or upload the file
6. Click **Import**

## Quick Installation (YAML Manifests)

If you prefer raw manifests:

### Step 1: Download Manifests

Download all manifests from the [Deployment Manifests](deployment-manifests.md) page:

- `01-namespace.yaml`
- `02-rbac.yaml`
- `03-loki-config.yaml`
- `04-loki-deployment.yaml`
- `05-promtail-config.yaml`
- `06-promtail-daemonset.yaml`
- `07-grafana-config.yaml`
- `08-grafana-deployment.yaml`
- `09-routes.yaml`

### Step 2: Update Configuration

Before deploying, update:

1. **Storage class** in PVC manifests (search for `storageClassName`)
2. **Admin password** in `07-grafana-config.yaml`
3. **Workflow namespaces** in `05-promtail-config.yaml`

### Step 3: Deploy

```bash
# Apply manifests in order
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

### Step 4: Verify

```bash
# Check pods
oc get pods -n sonataflow-observability

# All pods should be Running:
# loki-0
# promtail-xxxxx (one per node)
# grafana-xxxxxxxxxx-xxxxx
```

## Verify Log Collection

### Check Promtail is Scraping Logs

```bash
# Port-forward Promtail
oc port-forward -n sonataflow-observability ds/promtail 3101:3101 &

# Check targets
curl http://localhost:3101/targets | jq

# Check metrics
curl http://localhost:3101/metrics | grep promtail_targets_active_total
```

Expected output:
```
promtail_targets_active_total 5
```

### Check Loki is Receiving Logs

```bash
# Port-forward Loki
oc port-forward -n sonataflow-observability svc/loki 3100:3100 &

# Query Loki
curl -G -s "http://localhost:3100/loki/api/v1/query" \
  --data-urlencode 'query={job="sonataflow-workflows"}' \
  --data-urlencode 'limit=10' | jq '.data.result'
```

You should see log entries.

### Check Grafana Dashboard

1. Open Grafana
2. Navigate to **Explore**
3. Select **Loki** datasource
4. Enter query: `{job="sonataflow-workflows"}`
5. Click **Run Query**

You should see workflow logs.

## Common First-Time Issues

### Issue 1: No Logs in Loki

**Check Promtail is finding pods:**

```bash
oc logs -n sonataflow-observability ds/promtail --tail=50
```

Look for:
```
level=info msg="Successfully tailed" ...
```

**Verify pod labels:**

```bash
# Check SonataFlow pods have the right label
oc get pods -n sonataflow-infra --show-labels | grep sonataflow.org/workflow-app
```

If no output, Promtail won't find the pods. Check your Promtail configuration.

### Issue 2: Storage Issues

**Check storage class exists:**

```bash
oc get storageclass
```

**Update PVCs to use available storage class:**

```bash
# For Helm
# Edit loki-stack-values.yaml and change storageClassName

# For YAML manifests
# Edit each PVC's storageClassName field
```

### Issue 3: Grafana Can't Connect to Loki

**Test connectivity:**

```bash
oc exec -it -n sonataflow-observability deployment/grafana -- \
  wget -qO- http://loki:3100/ready
```

Expected: `ready`

If it fails, check:
1. Loki pod is running
2. Loki service exists
3. Network policies allow traffic

### Issue 4: Permission Denied for Promtail

**Apply SecurityContextConstraints:**

```bash
oc adm policy add-scc-to-user promtail-scc \
  -z promtail \
  -n sonataflow-observability
```

## Example Queries to Try

Once everything is running, try these queries in Grafana Explore:

### Get all workflow logs
```logql
{job="sonataflow-workflows"}
```

### Get errors only
```logql
{job="sonataflow-workflows", level="ERROR"}
```

### Get logs for specific workflow
```logql
{job="sonataflow-workflows", workflow_name="onboarding"}
```

### Get logs for specific process instance
```logql
{job="sonataflow-workflows", processInstanceId="abc-123"}
```

### Parse JSON and format
```logql
{job="sonataflow-workflows"}
| json
| line_format "{{.timestamp}} [{{.level}}] {{.state}}: {{.message}}"
```

See [LogQL Queries](logql-queries.md) for more examples.

## Next Steps

### 1. Import the Dashboard

Follow the [Grafana Dashboard](grafana-dashboard.md) guide to import the pre-built dashboard with:
- Process instance timeline
- Error tracking
- Performance metrics
- State distribution

### 2. Configure Alerts

Set up alerts for:
- High error rates
- Failed workflows
- Long-running processes

See [Alerting Configuration](grafana-dashboard.md#alerting-configuration).

### 3. Optimize for Production

Review:
- [Helm Values](helm-values.md#configuration-options) for resource tuning
- [Promtail Configuration](promtail-config.md#advanced-features) for advanced log processing
- [Deployment Manifests](deployment-manifests.md#security-hardening) for security hardening

### 4. Learn LogQL

Master log querying with [LogQL Query Examples](logql-queries.md):
- Filtering by process instance
- Correlating with traces
- Performance analysis
- Audit trails

## Production Checklist

Before going to production:

- [ ] Change default Grafana password
- [ ] Configure proper storage class and sizes
- [ ] Set up log retention policies
- [ ] Configure resource limits based on cluster size
- [ ] Enable TLS for internal communication
- [ ] Set up backup for Loki data
- [ ] Configure alerts for critical errors
- [ ] Document custom dashboards
- [ ] Train team on LogQL queries
- [ ] Set up access control (RBAC)

## Monitoring the Observability Stack

Keep an eye on the stack itself:

```bash
# Check Loki disk usage
oc exec -it -n sonataflow-observability loki-0 -- df -h /data

# Check Promtail metrics
oc port-forward -n sonataflow-observability ds/promtail 3101:3101
curl http://localhost:3101/metrics | grep promtail

# Monitor Grafana logs
oc logs -n sonataflow-observability deployment/grafana --tail=100 -f
```

## Getting Help

If you run into issues:

1. Check the [Troubleshooting](../troubleshooting.md) guide
2. Review component logs:
   ```bash
   oc logs -n sonataflow-observability <pod-name>
   ```
3. Check events:
   ```bash
   oc get events -n sonataflow-observability --sort-by='.lastTimestamp'
   ```
4. Verify RBAC permissions:
   ```bash
   oc auth can-i list pods --as=system:serviceaccount:sonataflow-observability:promtail
   ```

## Clean Up (for Testing)

To remove everything:

```bash
# For Helm installation
helm uninstall loki-stack -n sonataflow-observability
oc delete namespace sonataflow-observability

# For YAML installation
oc delete namespace sonataflow-observability
oc delete clusterrole promtail
oc delete clusterrolebinding promtail
oc delete scc promtail-scc
```

## Reference

- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [Promtail Documentation](https://grafana.com/docs/loki/latest/clients/promtail/)
- [Grafana Documentation](https://grafana.com/docs/grafana/latest/)
- [LogQL Documentation](https://grafana.com/docs/loki/latest/logql/)
- [SonataFlow Documentation](https://sonataflow.org/serverlessworkflow/latest/)

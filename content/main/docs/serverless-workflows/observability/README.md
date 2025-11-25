# SonataFlow Observability with PLG Stack

This directory contains comprehensive, production-ready documentation for implementing observability in SonataFlow workflows using the PLG (Promtail + Loki + Grafana) stack on OpenShift.

## Documentation Structure

```
observability/
├── _index.md                    # Overview and introduction
├── quick-start.md               # 15-minute quick start guide
├── helm-values.md               # Helm chart configuration
├── promtail-config.md           # Advanced Promtail setup
├── grafana-dashboard.md         # Dashboard JSON and customization
├── logql-queries.md             # Query examples and reference
└── deployment-manifests.md      # Complete YAML manifests
```

## Quick Links

- **New to PLG?** Start with [Quick Start Guide](quick-start.md)
- **Using Helm?** See [Helm Values](helm-values.md)
- **Need YAML?** Check [Deployment Manifests](deployment-manifests.md)
- **Want to query logs?** Read [LogQL Queries](logql-queries.md)
- **Setting up alerts?** Visit [Grafana Dashboard](grafana-dashboard.md)

## What You'll Get

### 1. Complete Deployment Solution
- Helm chart values for one-command deployment
- Raw Kubernetes/OpenShift YAML manifests
- RBAC and security configurations
- Network policies and resource limits

### 2. Production-Ready Configurations
- Non-root containers with SecurityContextConstraints
- Persistent storage for logs and dashboards
- TLS-enabled routes
- Resource quotas and limits
- Health checks and liveness probes

### 3. SonataFlow Integration
- Automatic workflow pod discovery
- JSON log parsing with MDC extraction
- Process instance tracking
- Trace correlation (OpenTelemetry compatible)
- Workflow state monitoring
- Platform services monitoring

### 4. Comprehensive Monitoring
- Pre-built Grafana dashboard with 10+ panels
- 50+ LogQL query examples
- Process instance timeline view
- Error tracking and alerting
- Performance metrics
- Audit trail capabilities

### 5. Documentation
- Step-by-step deployment guides
- Troubleshooting procedures
- Security hardening checklist
- Performance optimization tips
- Real-world examples

## Features

| Feature | Description |
|---------|-------------|
| **Automatic Discovery** | Promtail automatically finds SonataFlow pods using labels |
| **JSON Parsing** | Extracts processInstanceId, traceId, spanId from logs |
| **Kubernetes Metadata** | Adds namespace, pod, container labels automatically |
| **Multi-Namespace** | Monitor workflows across multiple namespaces |
| **Trace Correlation** | Link logs to distributed traces |
| **State Tracking** | Visualize workflow state transitions |
| **Error Detection** | Real-time error monitoring with alerts |
| **Performance Metrics** | Track workflow duration and throughput |
| **Audit Logging** | Complete audit trail for compliance |
| **Custom Queries** | 50+ example LogQL queries |

## Deployment Options

### Option 1: Helm (Quick Start)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki-stack grafana/loki-stack \
  --namespace sonataflow-observability \
  --values loki-stack-values.yaml
```

**Time to deploy**: ~5 minutes  
**Best for**: Development, quick testing, proof-of-concept  
**Pros**: Automated, easy upgrades  
**Cons**: Less customization

### Option 2: YAML Manifests (Production)

```bash
oc apply -f 01-namespace.yaml
oc apply -f 02-rbac.yaml
# ... (10 manifest files)
```

**Time to deploy**: ~15 minutes  
**Best for**: Production, GitOps workflows  
**Pros**: Full control, GitOps-friendly, easier audit  
**Cons**: More manual steps

## Quick Start (5 Minutes)

1. **Deploy stack**:
   ```bash
   helm install loki-stack grafana/loki-stack -n sonataflow-observability -f loki-stack-values.yaml
   ```

2. **Create route**:
   ```bash
   oc create route edge grafana --service=loki-stack-grafana -n sonataflow-observability
   ```

3. **Access Grafana**:
   ```bash
   echo "URL: https://$(oc get route grafana -n sonataflow-observability -o jsonpath='{.spec.host}')"
   echo "Password: $(oc get secret loki-stack-grafana -n sonataflow-observability -o jsonpath='{.data.admin-password}' | base64 -d)"
   ```

4. **Import dashboard**: Copy JSON from `grafana-dashboard.md`

5. **Run query**: `{job="sonataflow-workflows"}`

## File Descriptions

### _index.md
Overview page with architecture, features, and prerequisites.

### quick-start.md
Get started in 15 minutes with both Helm and YAML installation methods. Includes verification steps and common issues.

### helm-values.md
Complete `values.yaml` for the Grafana Loki Helm chart with:
- Loki configuration (storage, retention, limits)
- Promtail with SonataFlow pod discovery
- Grafana with pre-configured datasource
- OpenShift-specific security contexts

### promtail-config.md
Advanced Promtail configuration including:
- Standalone ConfigMap and DaemonSet
- JSON log parsing pipelines
- MDC field extraction
- Multiple scrape jobs (workflows, platform services, operator)
- Multi-line exception handling
- Sensitive data redaction

### grafana-dashboard.md
Production-ready dashboard with:
- Complete JSON (copy-paste ready)
- 10 visualization panels
- 6 dashboard variables
- Customization examples
- Alerting configuration

### logql-queries.md
Comprehensive query reference with:
- Common use cases (50+ examples)
- Advanced queries (lifecycle, performance, errors)
- Real-world examples (SLA, debugging, capacity planning)
- Alerting queries
- Optimization tips

### deployment-manifests.md
Complete Kubernetes/OpenShift manifests:
- 10 YAML files (namespace, RBAC, configs, deployments)
- SecurityContextConstraints
- NetworkPolicies
- ResourceQuotas
- Step-by-step deployment guide

## Examples Included

### LogQL Queries
- Get all logs for a process instance
- Find errors in a workflow
- Correlate logs with traces
- Calculate workflow duration
- Track state transitions
- Monitor error rates
- Detect anomalies
- Audit user actions

### Dashboard Panels
- Workflow log volume
- Error gauge
- Active process instances
- Log level distribution
- Recent errors table
- Process timeline
- State distribution pie chart
- Pod performance

### Pipeline Stages
- JSON parsing
- Timestamp extraction
- Label creation
- Structured metadata
- Metrics extraction
- Multi-line handling
- Redaction
- Conditional processing

## Use Cases

### Development
- Debug workflow executions
- Trace request flows
- Identify bottlenecks
- Test error handling

### Operations
- Monitor workflow health
- Track performance metrics
- Detect failures early
- Capacity planning

### Security & Compliance
- Audit trail of executions
- User activity tracking
- Data access logs
- Compliance reporting

### Business Intelligence
- Workflow completion rates
- SLA compliance
- Process duration analysis
- Error trend analysis

## Requirements

- OpenShift 4.12+
- SonataFlow Operator installed
- Cluster admin access (for initial setup)
- Storage provisioner for PVCs
- Helm 3.x (for Helm installation)

## Support

For issues and questions:
1. Check [Troubleshooting](../troubleshooting.md) guide
2. Review component logs
3. Verify RBAC permissions
4. Check network policies
5. Validate storage configuration

## Contributing

To improve this documentation:
1. Test configurations in your environment
2. Report issues or gaps
3. Suggest improvements
4. Share custom dashboards or queries

## License

Documentation follows the repository's license.

## Credits

Based on:
- [Grafana Loki](https://grafana.com/oss/loki/)
- [Promtail](https://grafana.com/docs/loki/latest/clients/promtail/)
- [Grafana](https://grafana.com/oss/grafana/)
- [SonataFlow](https://sonataflow.org/)

## Version

Documentation version: 1.0  
Last updated: 2025-11-24  
Compatible with:
- Loki 2.9.3
- Promtail 2.9.3
- Grafana 10.2.2
- SonataFlow 1.5+

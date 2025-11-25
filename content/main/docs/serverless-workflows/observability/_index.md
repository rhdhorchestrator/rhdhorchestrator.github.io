---
title: Observability with PLG Stack
date: "2025-11-24"
weight: 110
---

# Observability for SonataFlow Workflows

This section provides production-ready configurations for implementing observability in SonataFlow workflows using the PLG (Promtail + Loki + Grafana) stack on OpenShift.

## Overview

The PLG stack provides comprehensive log aggregation, querying, and visualization for SonataFlow workflows:

- **Promtail**: Discovers and scrapes logs from SonataFlow workflow pods
- **Loki**: Stores and indexes logs efficiently with labels
- **Grafana**: Visualizes logs and provides dashboards for workflow monitoring

## Features

- Automatic discovery of SonataFlow workflow pods
- JSON log parsing with MDC context (processInstanceId, traceId, spanId)
- Kubernetes metadata extraction (labels, annotations, pod info)
- Trace correlation and workflow performance monitoring
- Error tracking and alerting
- Process instance timeline visualization

## Contents

1. [Helm Values Configuration](helm-values.md) - Complete Helm chart configuration for Loki stack
2. [Promtail Configuration](promtail-config.md) - Advanced Promtail configuration for log scraping
3. [Grafana Dashboard](grafana-dashboard.md) - Pre-built dashboard for workflow monitoring
4. [LogQL Queries](logql-queries.md) - Query examples for common use cases
5. [Deployment Manifests](deployment-manifests.md) - YAML manifests for RBAC and resources

## Quick Start

1. Deploy the PLG stack using the Helm values
2. Configure Promtail to scrape SonataFlow pods
3. Import the Grafana dashboard
4. Start querying logs with LogQL

## Prerequisites

- OpenShift 4.12+
- SonataFlow Operator installed
- Cluster admin access (for initial setup)
- Helm 3.x

## Architecture

```
┌─────────────────┐
│ SonataFlow Pods │──┐
│  (JSON Logs)    │  │
└─────────────────┘  │
                     │ Scrape
┌─────────────────┐  │
│ SonataFlow Pods │──┤
└─────────────────┘  │
                     ▼
              ┌──────────┐      ┌──────┐      ┌─────────┐
              │ Promtail │─────▶│ Loki │◀────▶│ Grafana │
              └──────────┘      └──────┘      └─────────┘
              (Collector)      (Storage)     (Visualization)
```

## Log Format

SonataFlow workflows produce JSON structured logs with the following format:

```json
{
  "timestamp": "2025-11-24T10:30:45.123Z",
  "level": "INFO",
  "logger": "org.kie.kogito.workflow",
  "message": "Workflow step completed",
  "MDC": {
    "processInstanceId": "abc123-456-789",
    "traceId": "1a2b3c4d5e6f7g8h",
    "spanId": "9i0j1k2l",
    "workflowId": "onboarding",
    "state": "ApprovalStep"
  },
  "thread": "executor-thread-1",
  "exception": null
}
```

## Security Considerations

All configurations in this section follow OpenShift security best practices:

- Non-root containers with restricted SecurityContextConstraints
- Service accounts with minimal RBAC permissions
- TLS encryption for data in transit
- Namespace isolation
- Resource limits and quotas

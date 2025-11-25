---
title: Grafana Dashboard
date: "2025-11-24"
weight: 3
---

# Grafana Dashboard for SonataFlow Workflows

This page provides a complete, production-ready Grafana dashboard for monitoring SonataFlow workflows with comprehensive visualizations and metrics.

## Dashboard Overview

The dashboard includes:

- **Process Instance Timeline**: Visual timeline of workflow executions
- **Error Tracking**: Real-time error monitoring and alerting
- **Trace Correlation**: Link logs to distributed traces
- **Performance Metrics**: Workflow duration, throughput, and SLA compliance
- **State Transitions**: Visual flow of workflow state changes
- **Resource Usage**: Pod and container resource monitoring

## Import Dashboard

### Method 1: Using Grafana UI

1. Open Grafana (get URL from OpenShift route)
2. Navigate to **Dashboards** → **Import**
3. Paste the JSON below or upload the file
4. Select the Loki datasource
5. Click **Import**

### Method 2: Using ConfigMap

```bash
# Create the dashboard JSON file
cat > sonataflow-dashboard.json << 'EOF'
[Paste JSON from below]
EOF

# Create ConfigMap
oc create configmap grafana-sonataflow-dashboards \
  -n sonataflow-observability \
  --from-file=sonataflow-dashboard.json

# Label for auto-discovery
oc label configmap grafana-sonataflow-dashboards \
  -n sonataflow-observability \
  grafana_dashboard=1

# Restart Grafana to pick up the dashboard
oc rollout restart deployment/grafana -n sonataflow-observability
```

## Complete Dashboard JSON

```json
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": {
          "type": "datasource",
          "uid": "grafana"
        },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      },
      {
        "datasource": {
          "type": "loki",
          "uid": "loki"
        },
        "enable": true,
        "expr": "{job=\"sonataflow-workflows\", level=\"ERROR\"}",
        "iconColor": "red",
        "name": "Workflow Errors",
        "tagKeys": "workflow_name,processInstanceId",
        "textFormat": "{{ message }}",
        "titleFormat": "Error in {{ workflow_name }}"
      }
    ]
  },
  "description": "Comprehensive monitoring dashboard for SonataFlow workflows with log aggregation, error tracking, and performance metrics",
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 1,
  "id": null,
  "links": [
    {
      "asDropdown": false,
      "icon": "external link",
      "includeVars": true,
      "keepTime": true,
      "tags": ["sonataflow"],
      "targetBlank": true,
      "title": "SonataFlow Dashboards",
      "type": "dashboards"
    }
  ],
  "liveNow": false,
  "panels": [
    {
      "datasource": {
        "type": "loki",
        "uid": "loki"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": {
              "tooltip": false,
              "viz": false,
              "legend": false
            },
            "lineInterpolation": "smooth",
            "lineWidth": 2,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "short"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 0
      },
      "id": 1,
      "options": {
        "legend": {
          "calcs": ["lastNotNull", "max"],
          "displayMode": "table",
          "placement": "right",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "desc"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "loki"
          },
          "expr": "sum by (workflow_name) (count_over_time({job=\"sonataflow-workflows\", workflow_name=~\"$workflow_name\"} [$__interval]))",
          "legendFormat": "{{ workflow_name }}",
          "refId": "A"
        }
      ],
      "title": "Workflow Log Volume by Workflow",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "loki"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "yellow",
                "value": 10
              },
              {
                "color": "red",
                "value": 50
              }
            ]
          },
          "unit": "short"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 6,
        "x": 12,
        "y": 0
      },
      "id": 2,
      "options": {
        "orientation": "auto",
        "reduceOptions": {
          "values": false,
          "calcs": ["lastNotNull"],
          "fields": ""
        },
        "showThresholdLabels": false,
        "showThresholdMarkers": true,
        "text": {}
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "loki"
          },
          "expr": "sum(count_over_time({job=\"sonataflow-workflows\", level=\"ERROR\", workflow_name=~\"$workflow_name\"} [$__range]))",
          "refId": "A"
        }
      ],
      "title": "Total Errors",
      "type": "gauge"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "loki"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "short"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 6,
        "x": 18,
        "y": 0
      },
      "id": 3,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "values": false,
          "calcs": ["lastNotNull"],
          "fields": ""
        },
        "text": {},
        "textMode": "auto"
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "loki"
          },
          "expr": "count(count by (processInstanceId) (count_over_time({job=\"sonataflow-workflows\", workflow_name=~\"$workflow_name\"} [$__range])))",
          "refId": "A"
        }
      ],
      "title": "Active Process Instances",
      "type": "stat"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "loki"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "bars",
            "fillOpacity": 80,
            "gradientMode": "none",
            "hideFrom": {
              "tooltip": false,
              "viz": false,
              "legend": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "normal"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "short"
        },
        "overrides": [
          {
            "matcher": {
              "id": "byName",
              "options": "ERROR"
            },
            "properties": [
              {
                "id": "color",
                "value": {
                  "fixedColor": "red",
                  "mode": "fixed"
                }
              }
            ]
          },
          {
            "matcher": {
              "id": "byName",
              "options": "WARN"
            },
            "properties": [
              {
                "id": "color",
                "value": {
                  "fixedColor": "yellow",
                  "mode": "fixed"
                }
              }
            ]
          },
          {
            "matcher": {
              "id": "byName",
              "options": "INFO"
            },
            "properties": [
              {
                "id": "color",
                "value": {
                  "fixedColor": "blue",
                  "mode": "fixed"
                }
              }
            ]
          }
        ]
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 8
      },
      "id": 4,
      "options": {
        "legend": {
          "calcs": ["sum"],
          "displayMode": "table",
          "placement": "right",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "desc"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "loki"
          },
          "expr": "sum by (level) (count_over_time({job=\"sonataflow-workflows\", workflow_name=~\"$workflow_name\"} [$__interval]))",
          "legendFormat": "{{ level }}",
          "refId": "A"
        }
      ],
      "title": "Log Levels Over Time",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "loki"
      },
      "fieldConfig": {
        "defaults": {
          "custom": {
            "align": "auto",
            "cellOptions": {
              "type": "auto"
            },
            "inspect": false
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "color": {
            "mode": "thresholds"
          }
        },
        "overrides": [
          {
            "matcher": {
              "id": "byName",
              "options": "level"
            },
            "properties": [
              {
                "id": "custom.cellOptions",
                "value": {
                  "type": "color-background"
                }
              },
              {
                "id": "mappings",
                "value": [
                  {
                    "options": {
                      "ERROR": {
                        "color": "red",
                        "index": 0
                      },
                      "WARN": {
                        "color": "yellow",
                        "index": 1
                      },
                      "INFO": {
                        "color": "blue",
                        "index": 2
                      },
                      "DEBUG": {
                        "color": "green",
                        "index": 3
                      }
                    },
                    "type": "value"
                  }
                ]
              }
            ]
          }
        ]
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 8
      },
      "id": 5,
      "options": {
        "showHeader": true,
        "cellHeight": "sm",
        "footer": {
          "show": false,
          "reducer": ["sum"],
          "countRows": false,
          "fields": ""
        },
        "frameIndex": 0
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "loki"
          },
          "expr": "{job=\"sonataflow-workflows\", level=\"ERROR\", workflow_name=~\"$workflow_name\"} |~ \"$search\"",
          "refId": "A"
        }
      ],
      "title": "Recent Errors",
      "transformations": [
        {
          "id": "extractFields",
          "options": {
            "source": "labels"
          }
        }
      ],
      "type": "table"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "loki"
      },
      "gridPos": {
        "h": 10,
        "w": 24,
        "x": 0,
        "y": 16
      },
      "id": 6,
      "options": {
        "dedupStrategy": "none",
        "enableLogDetails": true,
        "prettifyLogMessage": true,
        "showCommonLabels": false,
        "showLabels": false,
        "showTime": true,
        "sortOrder": "Descending",
        "wrapLogMessage": false
      },
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "loki"
          },
          "expr": "{job=\"sonataflow-workflows\", workflow_name=~\"$workflow_name\", level=~\"$log_level\", processInstanceId=~\"$process_instance\"} |~ \"$search\"",
          "refId": "A"
        }
      ],
      "title": "Workflow Logs",
      "type": "logs"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "loki"
      },
      "fieldConfig": {
        "defaults": {
          "custom": {
            "align": "auto",
            "cellOptions": {
              "type": "auto"
            },
            "inspect": false
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 26
      },
      "id": 7,
      "options": {
        "showHeader": true,
        "cellHeight": "sm",
        "footer": {
          "show": false,
          "reducer": ["sum"],
          "countRows": false,
          "fields": ""
        }
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "loki"
          },
          "expr": "{job=\"sonataflow-workflows\", processInstanceId=\"$process_instance\"} | json | line_format \"{{.timestamp}} [{{.level}}] {{.state}}: {{.message}}\"",
          "refId": "A"
        }
      ],
      "title": "Process Instance Timeline - $process_instance",
      "transformations": [],
      "type": "table"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "loki"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "hideFrom": {
              "tooltip": false,
              "viz": false,
              "legend": false
            }
          },
          "mappings": []
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 26
      },
      "id": 8,
      "options": {
        "legend": {
          "displayMode": "table",
          "placement": "right",
          "showLegend": true,
          "values": ["value", "percent"]
        },
        "pieType": "donut",
        "tooltip": {
          "mode": "single",
          "sort": "none"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "loki"
          },
          "expr": "sum by (state) (count_over_time({job=\"sonataflow-workflows\", workflow_name=~\"$workflow_name\"} | json | state != \"\" [$__range]))",
          "legendFormat": "{{ state }}",
          "refId": "A"
        }
      ],
      "title": "Workflow State Distribution",
      "type": "piechart"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "loki"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 0,
            "gradientMode": "none",
            "hideFrom": {
              "tooltip": false,
              "viz": false,
              "legend": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 24,
        "x": 0,
        "y": 34
      },
      "id": 9,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "single",
          "sort": "none"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "loki"
          },
          "expr": "sum by (namespace, pod) (rate({job=\"sonataflow-workflows\", workflow_name=~\"$workflow_name\"} [$__interval]))",
          "legendFormat": "{{ namespace }}/{{ pod }}",
          "refId": "A"
        }
      ],
      "title": "Log Rate by Pod",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "loki"
      },
      "fieldConfig": {
        "defaults": {
          "custom": {
            "align": "auto",
            "cellOptions": {
              "type": "auto"
            },
            "inspect": false
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 24,
        "x": 0,
        "y": 42
      },
      "id": 10,
      "options": {
        "showHeader": true,
        "cellHeight": "sm",
        "footer": {
          "show": true,
          "reducer": ["count"],
          "countRows": false,
          "fields": ""
        }
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "loki"
          },
          "expr": "topk(20, sum by (processInstanceId, workflow_name) (count_over_time({job=\"sonataflow-workflows\", workflow_name=~\"$workflow_name\"} [$__range])))",
          "refId": "A"
        }
      ],
      "title": "Top 20 Active Process Instances",
      "transformations": [],
      "type": "table"
    }
  ],
  "refresh": "30s",
  "schemaVersion": 38,
  "style": "dark",
  "tags": ["sonataflow", "workflows", "observability", "loki"],
  "templating": {
    "list": [
      {
        "current": {
          "selected": false,
          "text": "Loki",
          "value": "Loki"
        },
        "hide": 0,
        "includeAll": false,
        "multi": false,
        "name": "datasource",
        "options": [],
        "query": "loki",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "type": "datasource"
      },
      {
        "current": {
          "selected": true,
          "text": ["All"],
          "value": ["$__all"]
        },
        "datasource": {
          "type": "loki",
          "uid": "loki"
        },
        "definition": "label_values(workflow_name)",
        "hide": 0,
        "includeAll": true,
        "multi": true,
        "name": "workflow_name",
        "options": [],
        "query": {
          "label": "workflow_name",
          "refId": "LokiVariableQueryEditor-VariableQuery",
          "stream": "",
          "type": 1
        },
        "refresh": 2,
        "regex": "",
        "skipUrlSync": false,
        "sort": 1,
        "type": "query"
      },
      {
        "current": {
          "selected": true,
          "text": ["All"],
          "value": ["$__all"]
        },
        "datasource": {
          "type": "loki",
          "uid": "loki"
        },
        "definition": "label_values({workflow_name=~\"$workflow_name\"}, namespace)",
        "hide": 0,
        "includeAll": true,
        "multi": true,
        "name": "namespace",
        "options": [],
        "query": {
          "label": "namespace",
          "refId": "LokiVariableQueryEditor-VariableQuery",
          "stream": "{workflow_name=~\"$workflow_name\"}",
          "type": 1
        },
        "refresh": 2,
        "regex": "",
        "skipUrlSync": false,
        "sort": 1,
        "type": "query"
      },
      {
        "current": {
          "selected": false,
          "text": "All",
          "value": "$__all"
        },
        "datasource": {
          "type": "loki",
          "uid": "loki"
        },
        "definition": "label_values({workflow_name=~\"$workflow_name\", namespace=~\"$namespace\"}, processInstanceId)",
        "hide": 0,
        "includeAll": true,
        "multi": false,
        "name": "process_instance",
        "options": [],
        "query": {
          "label": "processInstanceId",
          "refId": "LokiVariableQueryEditor-VariableQuery",
          "stream": "{workflow_name=~\"$workflow_name\", namespace=~\"$namespace\"}",
          "type": 1
        },
        "refresh": 2,
        "regex": "",
        "skipUrlSync": false,
        "sort": 1,
        "type": "query"
      },
      {
        "current": {
          "selected": true,
          "text": ["All"],
          "value": ["$__all"]
        },
        "hide": 0,
        "includeAll": true,
        "multi": true,
        "name": "log_level",
        "options": [
          {
            "selected": true,
            "text": "All",
            "value": "$__all"
          },
          {
            "selected": false,
            "text": "ERROR",
            "value": "ERROR"
          },
          {
            "selected": false,
            "text": "WARN",
            "value": "WARN"
          },
          {
            "selected": false,
            "text": "INFO",
            "value": "INFO"
          },
          {
            "selected": false,
            "text": "DEBUG",
            "value": "DEBUG"
          }
        ],
        "query": "ERROR,WARN,INFO,DEBUG",
        "queryValue": "",
        "skipUrlSync": false,
        "type": "custom"
      },
      {
        "current": {
          "selected": false,
          "text": "",
          "value": ""
        },
        "hide": 0,
        "name": "search",
        "options": [
          {
            "selected": true,
            "text": "",
            "value": ""
          }
        ],
        "query": "",
        "skipUrlSync": false,
        "type": "textbox",
        "label": "Search"
      }
    ]
  },
  "time": {
    "from": "now-6h",
    "to": "now"
  },
  "timepicker": {
    "refresh_intervals": ["5s", "10s", "30s", "1m", "5m", "15m", "30m", "1h", "2h", "1d"]
  },
  "timezone": "",
  "title": "SonataFlow Workflows Observability",
  "uid": "sonataflow-workflows",
  "version": 1,
  "weekStart": ""
}
```

## Dashboard Features

### 1. Variables

The dashboard includes several variables for filtering:

- **datasource**: Select the Loki datasource
- **workflow_name**: Filter by workflow name (multi-select)
- **namespace**: Filter by namespace (multi-select)
- **process_instance**: Filter by specific process instance ID
- **log_level**: Filter by log level (ERROR, WARN, INFO, DEBUG)
- **search**: Free-text search in log messages

### 2. Panels

#### Panel 1: Workflow Log Volume by Workflow
- **Type**: Time series
- **Query**: `sum by (workflow_name) (count_over_time({job="sonataflow-workflows", workflow_name=~"$workflow_name"} [$__interval]))`
- **Purpose**: Shows log volume trends per workflow

#### Panel 2: Total Errors
- **Type**: Gauge
- **Query**: `sum(count_over_time({job="sonataflow-workflows", level="ERROR", workflow_name=~"$workflow_name"} [$__range]))`
- **Purpose**: Quick view of total errors in selected time range

#### Panel 3: Active Process Instances
- **Type**: Stat
- **Query**: `count(count by (processInstanceId) (...))`
- **Purpose**: Number of unique process instances

#### Panel 4: Log Levels Over Time
- **Type**: Stacked bars
- **Purpose**: Visualize log level distribution over time

#### Panel 5: Recent Errors
- **Type**: Table
- **Purpose**: List of recent ERROR level logs

#### Panel 6: Workflow Logs
- **Type**: Logs panel
- **Purpose**: Main log viewer with full-text search

#### Panel 7: Process Instance Timeline
- **Type**: Table
- **Purpose**: Chronological view of a specific process instance

#### Panel 8: Workflow State Distribution
- **Type**: Pie chart
- **Purpose**: Shows distribution of workflow states

#### Panel 9: Log Rate by Pod
- **Type**: Time series
- **Purpose**: Identifies which pods are most active

#### Panel 10: Top 20 Active Process Instances
- **Type**: Table
- **Purpose**: Lists most active process instances by log count

### 3. Annotations

The dashboard includes automatic annotations for:
- Workflow errors (shown as red markers)
- Custom events (can be added via LogQL)

## Customization Examples

### Add Custom Panel for Workflow Duration

```json
{
  "datasource": {
    "type": "loki",
    "uid": "loki"
  },
  "fieldConfig": {
    "defaults": {
      "unit": "s"
    }
  },
  "targets": [
    {
      "expr": "{job=\"sonataflow-workflows\", workflow_name=~\"$workflow_name\"} | json | duration != \"\" | unwrap duration | quantile_over_time(0.95, [$__interval])",
      "legendFormat": "p95 {{ workflow_name }}",
      "refId": "A"
    }
  ],
  "title": "Workflow Duration (p95)",
  "type": "timeseries"
}
```

### Add SLA Compliance Panel

```json
{
  "datasource": {
    "type": "loki",
    "uid": "loki"
  },
  "targets": [
    {
      "expr": "(sum(count_over_time({job=\"sonataflow-workflows\", message=~\".*completed.*\"} [$__range])) - sum(count_over_time({job=\"sonataflow-workflows\", message=~\".*SLA.*exceeded.*\"} [$__range]))) / sum(count_over_time({job=\"sonataflow-workflows\", message=~\".*completed.*\"} [$__range])) * 100",
      "refId": "A"
    }
  ],
  "title": "SLA Compliance %",
  "type": "stat",
  "fieldConfig": {
    "defaults": {
      "unit": "percent",
      "thresholds": {
        "steps": [
          {"color": "red", "value": 0},
          {"color": "yellow", "value": 95},
          {"color": "green", "value": 99}
        ]
      }
    }
  }
}
```

### Add Trace Correlation Links

If you have Tempo for distributed tracing:

```json
{
  "datasource": {
    "type": "loki",
    "uid": "loki"
  },
  "fieldConfig": {
    "overrides": [
      {
        "matcher": {"id": "byName", "options": "traceId"},
        "properties": [
          {
            "id": "links",
            "value": [
              {
                "title": "View Trace",
                "url": "/explore?left={\"datasource\":\"tempo\",\"queries\":[{\"query\":\"${__value.raw}\"}]}"
              }
            ]
          }
        ]
      }
    ]
  }
}
```

## Alerting Configuration

### Create Alert Rule in Grafana

1. Navigate to **Alerting** → **Alert Rules**
2. Click **New Alert Rule**
3. Configure the alert:

**Alert Name**: High Error Rate in Workflows

**Query**:
```
sum(rate({job="sonataflow-workflows", level="ERROR"}[5m])) > 10
```

**Condition**:
- Reduce expression: Last
- Threshold: Above 10

**Notification**:
- Contact point: (configure email/Slack/webhook)
- Message: `Workflow {{ $labels.workflow_name }} has high error rate: {{ $value }} errors/sec`

### Example Alert Queries

#### Alert on Workflow Failure
```
count_over_time({job="sonataflow-workflows", message=~".*workflow failed.*"}[5m]) > 0
```

#### Alert on Long-Running Workflows
```
{job="sonataflow-workflows"} | json | duration > 300
```

#### Alert on Missing Process Instances
```
absent_over_time({job="sonataflow-workflows", workflow_name="critical-workflow"}[10m])
```

## Dashboard Export/Import

### Export Dashboard

```bash
# Using Grafana API
curl -H "Authorization: Bearer ${GRAFANA_API_KEY}" \
  http://grafana.sonataflow-observability/api/dashboards/uid/sonataflow-workflows \
  | jq '.dashboard' > sonataflow-dashboard.json
```

### Import Dashboard via API

```bash
# Create dashboard via API
curl -X POST \
  -H "Authorization: Bearer ${GRAFANA_API_KEY}" \
  -H "Content-Type: application/json" \
  -d @sonataflow-dashboard.json \
  http://grafana.sonataflow-observability/api/dashboards/db
```

## Performance Optimization

### 1. Limit Query Range

For large deployments, limit the query time range:

```json
{
  "time": {
    "from": "now-1h",
    "to": "now"
  }
}
```

### 2. Use Query Caching

Enable query result caching in Loki configuration:

```yaml
query_range:
  cache_results: true
  results_cache:
    cache:
      enable_fifocache: true
      fifocache:
        max_size_bytes: 500MB
```

### 3. Optimize Panel Queries

Use `$__interval` for dynamic time grouping:

```
sum by (workflow_name) (count_over_time({job="sonataflow-workflows"} [$__interval]))
```

## Troubleshooting

### Dashboard Shows "No Data"

1. Check Loki datasource connection
2. Verify logs are being ingested: See [LogQL Queries](logql-queries.md)
3. Check variable values (workflow_name, namespace)
4. Verify time range

### Slow Dashboard Performance

1. Reduce time range
2. Limit number of workflows in filter
3. Increase query timeout in Grafana settings
4. Enable caching in Loki

### Variables Not Populating

1. Check Loki datasource is selected
2. Verify label names match your log labels
3. Test query in Explore view first

## Next Steps

- [Learn LogQL Queries](logql-queries.md) for advanced queries
- [Configure Alerts](logql-queries.md#alerting) for proactive monitoring
- [View Deployment Manifests](deployment-manifests.md) for RBAC setup

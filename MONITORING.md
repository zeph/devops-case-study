# Monitoring & Observability Documentation

This document describes the monitoring and observability stack deployed for the Voize ML platform.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Monitoring Stack                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐  │
│  │  Prometheus │◄───│ServiceMonitor│◄───│  ML API / Backend API  │  │
│  │             │    └─────────────┘    │      /metrics           │  │
│  └──────┬──────┘                       └─────────────────────────┘  │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────┐    ┌─────────────┐                                 │
│  │   Grafana   │    │ Alertmanager│                                 │
│  │ (Dashboards)│    │  (Alerts)   │                                 │
│  └─────────────┘    └─────────────┘                                 │
└─────────────────────────────────────────────────────────────────────┘
```

## Components

### 1. Prometheus (Metrics Collection)
- **Chart**: `kube-prometheus-stack` from prometheus-community
- **Retention**: 7 days
- **Storage**: 10Gi PVC
- **Scrape interval**: 15s for application metrics

### 2. Grafana (Visualization)
- **Default credentials**: admin / admin
- **Pre-configured dashboards**: 3 custom dashboards + default Kubernetes dashboards
- **Auto-discovery**: Dashboards are loaded from ConfigMaps with `grafana_dashboard: "1"` label

### 3. Alertmanager (Alert Routing)
- Receives alerts from Prometheus
- Can be configured for Slack, PagerDuty, email, etc. (not configured in this setup)

## What We Monitor

### ML API Metrics
| Metric | Type | Purpose |
|--------|------|---------|
| `ml_api_requests_total` | Counter | Track request volume and error rates by endpoint/status |
| `ml_api_request_duration_seconds` | Histogram | Measure latency percentiles (p50, p95, p99) |
| `ml_api_predictions_total` | Counter | Track ML inference throughput |
| `ml_api_memory_bytes` | Gauge | Monitor memory usage for resource planning |

### Backend API Metrics
| Metric | Type | Purpose |
|--------|------|---------|
| `backend_api_requests_total` | Counter | Track request volume and error rates |
| `backend_api_request_duration_seconds` | Histogram | Measure API latency |
| `backend_api_db_connections_active` | Gauge | Monitor database connection pool health |
| `backend_api_db_queries_total` | Counter | Track database query success/failure rates |

### Infrastructure Metrics
- Pod restarts and crash loops
- Deployment replica availability
- Container CPU and memory usage vs limits
- Node-level metrics (via node-exporter)

## Dashboards

### 1. Voize Platform Overview (`voize-overview`)
**Purpose**: Single-pane-of-glass view for on-call engineers

**Key panels**:
- Service health status (UP/DOWN)
- Pod counts per service
- Firing alerts count
- Error rates and latency for both services
- Request rate trends

### 2. ML API Dashboard (`ml-api-dashboard`)
**Purpose**: Deep-dive into ML inference service performance

**Sections**:
- **Overview**: Healthy pods, error rate, P95 latency, RPS, predictions/min, memory
- **Request Metrics**: Request rate by endpoint and status code
- **Latency**: Percentile breakdown (p50/p95/p99) by endpoint and pod
- **Resources**: Memory usage and prediction throughput per pod

### 3. Backend API Dashboard (`backend-api-dashboard`)
**Purpose**: Deep-dive into document processing service

**Sections**:
- **Overview**: Healthy pods, error rate, P95 latency, RPS, DB connections, DB error rate
- **Request Metrics**: Request rate by endpoint and status code
- **Latency**: Percentile breakdown by endpoint and pod
- **Database**: Active connections per pod, query rate by status

## Alerting Rules

### Critical Alerts (Immediate Action Required)

| Alert | Condition | Description |
|-------|-----------|-------------|
| `MLAPIHighErrorRate` | >5% 5xx errors for 2min | ML API is returning too many errors |
| `MLAPIDown` | Service unreachable for 1min | ML API instance is not responding |
| `BackendAPIHighErrorRate` | >5% 5xx errors for 2min | Backend API is returning too many errors |
| `BackendAPIDown` | Service unreachable for 1min | Backend API instance is not responding |
| `BackendAPIDBQueryErrors` | DB query errors >0.1/s for 2min | Database connectivity issues |
| `PodCrashLooping` | >3 restarts in 1 hour | Pod is repeatedly crashing |

### Warning Alerts (Investigation Needed)

| Alert | Condition | Description |
|-------|-----------|-------------|
| `MLAPIHighLatency` | P95 >2s for 5min | ML inference is slow |
| `MLAPIHighMemoryUsage` | >200MB for 5min | Potential memory leak |
| `MLAPINoPredictions` | 0 predictions for 10min | No ML work being done |
| `BackendAPIHighLatency` | P95 >1s for 5min | API responses are slow |
| `BackendAPIHighDBConnections` | >8 connections for 5min | Connection pool exhaustion risk |
| `PodNotReady` | Pod not ready for 5min | Pod health check failing |
| `DeploymentReplicasMismatch` | Desired ≠ Available for 10min | Scaling or scheduling issues |
| `HighPodMemoryUsage` | >90% of limit for 5min | OOMKill risk |
| `HighPodCPUUsage` | >90% of limit for 5min | CPU throttling |

## Accessing the Stack

```bash
# Grafana (dashboards)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# Open http://localhost:3000 (admin/admin)

# Prometheus (metrics & queries)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Open http://localhost:9090

# Alertmanager (alert management)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093
# Open http://localhost:9093
```

## Tradeoffs & Future Improvements

### Current Tradeoffs

1. **No persistent Grafana storage**: Dashboard changes made in UI won't persist across restarts. All dashboards are defined as ConfigMaps (GitOps-friendly).

2. **Basic Alertmanager config**: Alerts are collected but not routed to external systems. In production, configure Slack/PagerDuty/email receivers.

3. **No log aggregation**: Focused on metrics only. For production, add Loki or similar for log correlation.

4. **Single Prometheus instance**: No HA setup. For production, consider Thanos or Prometheus HA pairs.

5. **Local storage**: Using local PVC. For production, use cloud-native storage with proper backup.

### With More Time

1. **Add Loki for logs**: Correlate logs with metrics for faster debugging
2. **Configure alert routing**: Set up Slack/PagerDuty integration
3. **Add SLO dashboards**: Define and track SLIs/SLOs for the services
4. **Distributed tracing**: Add Jaeger/Tempo for request tracing across services
5. **PostgreSQL monitoring**: Add postgres-exporter for database metrics
6. **Runbooks**: Link alerts to runbook documentation
7. **Recording rules**: Pre-compute expensive queries for dashboard performance

## GitOps Structure

```
infrastructure/
├── controllers/
│   └── monitoring/
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── helm-repository.yaml      # prometheus-community charts
│       └── helm-release.yaml          # kube-prometheus-stack
└── configs/
    └── monitoring/
        ├── kustomization.yaml
        ├── servicemonitor-ml-api.yaml
        ├── servicemonitor-backend-api.yaml
        ├── prometheus-rules.yaml
        ├── grafana-dashboard-overview.yaml
        ├── grafana-dashboard-ml-api.yaml
        └── grafana-dashboard-backend-api.yaml
```

The monitoring stack is deployed in two phases:
1. **Controllers**: Installs the kube-prometheus-stack (Prometheus, Grafana, Alertmanager, CRDs)
2. **Configs**: Deploys ServiceMonitors, PrometheusRules, and Grafana dashboards

This separation ensures CRDs are available before custom resources are created.

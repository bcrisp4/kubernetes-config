# Metrics Collection Architecture

This document describes the metrics collection pipeline for the nbg1-prod1 cluster.

## Overview

Metrics are collected from Kubernetes pods using OpenTelemetry Collector and stored in Grafana Mimir for long-term storage and querying via Grafana.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Kubernetes Cluster                              │
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                       │
│  │   Pod with   │  │   Pod with   │  │   Pod with   │                       │
│  │   /metrics   │  │   /metrics   │  │   /metrics   │   ... more pods       │
│  │  (annotated) │  │ (metrics port)│  │  (annotated) │                       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                       │
│         │                 │                 │                                │
│         └────────────┬────┴─────────────────┘                                │
│                      │ Prometheus scrape                                     │
│                      ▼                                                       │
│         ┌────────────────────────┐                                          │
│         │   otel-metrics         │                                          │
│         │   (DaemonSet)          │                                          │
│         │                        │                                          │
│         │  • Prometheus Receiver │                                          │
│         │  • Kubernetes SD       │                                          │
│         │  • Node-local scraping │                                          │
│         └───────────┬────────────┘                                          │
│                     │ OTLP/HTTP                                             │
│                     ▼                                                       │
│         ┌────────────────────────┐      ┌────────────────────────┐         │
│         │   Mimir Gateway        │─────▶│   Mimir Distributors   │         │
│         │   /otlp/v1/metrics     │      │   (write path)         │         │
│         └────────────────────────┘      └───────────┬────────────┘         │
│                                                     │                       │
│                                                     ▼                       │
│                                         ┌────────────────────────┐         │
│                                         │   Kafka (ingest)       │         │
│                                         │   via Strimzi          │         │
│                                         └───────────┬────────────┘         │
│                                                     │                       │
│                                                     ▼                       │
│                                         ┌────────────────────────┐         │
│                                         │   Mimir Ingesters      │         │
│                                         │   (write to S3)        │         │
│                                         └───────────┬────────────┘         │
│                                                     │                       │
│                                                     ▼                       │
│                                         ┌────────────────────────┐         │
│                                         │   S3 Object Storage    │         │
│                                         │   (Hetzner)            │         │
│                                         └────────────────────────┘         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Components

### otel-metrics (OpenTelemetry Collector)

**Deployment**: DaemonSet (one pod per node)

**Purpose**: Scrapes Prometheus metrics from pods and forwards to Mimir via OTLP.

**Key Features**:
- Node-local scraping: Each collector only scrapes pods on its own node
- Dual discovery: Supports both annotation-based and port-name-based discovery
- OTLP export: Uses native OTLP protocol to Mimir (not Prometheus remote write)

### Mimir

**Deployment**: Distributed mode with Kafka ingest storage

**Components**:
- Gateway: Entry point for writes and queries
- Distributors: Receive and distribute incoming metrics
- Ingesters: Buffer and write to long-term storage
- Queriers: Query metrics from storage
- Store Gateway: Query historical data from S3
- Compactor: Compact and deduplicate blocks

**Storage**:
- Ingest: Strimzi Kafka (KRaft mode, 3 replicas)
- Long-term: S3-compatible object storage (Hetzner)

## Service Discovery

### Method 1: Annotation-based (Recommended)

Pods opt-in to metrics scraping using standard Prometheus annotations:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"      # optional, defaults to container port
    prometheus.io/path: "/metrics"  # optional, defaults to /metrics
```

**Currently annotated**:
- cert-manager (3 pods)
- Istio components (19 pods)

### Method 2: Port-name-based (Fallback)

Pods without annotations are discovered if they have a port with "metrics" in the name:

```yaml
spec:
  containers:
    - ports:
        - name: metrics       # or http-metrics, prom-metrics, etc.
          containerPort: 8080
```

**Discovered via port name**:
- Mimir (30 pods)
- Loki (21 pods)
- ArgoCD (8 pods)
- External Secrets (3 pods)
- And more...

### Opting Out

To prevent scraping, set the annotation explicitly:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "false"
```

## Data Flow

1. **Scrape**: otel-metrics scrapes /metrics endpoints every 30s
2. **Transform**: Metrics are enriched with Kubernetes metadata (namespace, pod, node)
3. **Batch**: Metrics are batched for efficient transmission
4. **Export**: Sent to Mimir via OTLP HTTP at `/otlp/v1/metrics`
5. **Ingest**: Mimir distributes to Kafka, then to ingesters
6. **Store**: Ingesters periodically flush blocks to S3
7. **Query**: Grafana queries Mimir via Prometheus-compatible API

## Labels

All metrics include these labels:

| Label | Source | Example |
|-------|--------|---------|
| `namespace` | Kubernetes | `mimir` |
| `pod` | Kubernetes | `mimir-ingester-0` |
| `node` | Kubernetes | `node-1` |
| `cluster` | Static | `nbg1-prod1` |
| `job` | Scrape config | `kubernetes-pods-annotations` |

Additional labels from pod labels are preserved with their original names.

## Querying

### Grafana Datasource

```
URL: http://mimir-gateway.mimir.svc.cluster.local/prometheus
Type: Prometheus
```

### Example PromQL Queries

```promql
# CPU usage by namespace
sum by (namespace) (rate(container_cpu_usage_seconds_total[5m]))

# Memory usage by pod
container_memory_usage_bytes{namespace="mimir"}

# HTTP request rate
sum by (handler) (rate(http_requests_total{namespace="grafana"}[5m]))
```

## Istio Integration

The otel-metrics namespace is added to the Istio ambient mesh. Authorization policies allow:
- otel-metrics → mimir (for OTLP export)
- Internal mesh traffic

## Resource Allocation

| Component | CPU Request | Memory Request | Memory Limit |
|-----------|-------------|----------------|--------------|
| otel-metrics | 100m | 128Mi | 512Mi |

## Scaling Considerations

- **Horizontal**: DaemonSet automatically scales with cluster nodes
- **Cardinality**: Monitor `prometheus_tsdb_head_series` for high cardinality
- **Scrape interval**: 30s default, adjustable per-job if needed

## Troubleshooting

### Check scrape targets

```bash
# View discovered targets
kubectl exec -n otel-metrics <pod> -- wget -qO- localhost:8888/metrics | grep scrape

# Check collector logs
kubectl logs -n otel-metrics -l app.kubernetes.io/name=opentelemetry-collector
```

### Verify Mimir ingestion

```bash
# Check distributor metrics
kubectl exec -n mimir deployment/mimir-distributor -- wget -qO- localhost:8080/metrics | grep cortex_distributor
```

### Test OTLP endpoint

```bash
kubectl exec -n otel-metrics <pod> -- wget -qO- http://mimir-gateway.mimir.svc.cluster.local/otlp/v1/metrics
```

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Kubernetes configuration repository containing Helm wrapper charts for deploying applications across multiple clusters. Each cluster has its own directory with per-application Helm charts.

## Repository Structure

```
<cluster-name>/
  <application>/
    Chart.yaml        # Helm chart with upstream dependency
    values.yaml       # Default values for the chart
    overrides/        # Empty or cluster-specific overrides
      values.yaml
    templates/        # Custom templates (secrets, CRDs, etc.)
    charts/           # Downloaded chart dependencies (gitignored)
```

### Clusters
- `nbg1-prod1` - Production cluster (Hetzner NBG1 datacenter)
- `raspberrypi1` - Raspberry Pi cluster

## Chart Pattern

Each application uses a wrapper chart pattern:
1. `Chart.yaml` declares an upstream Helm chart dependency (e.g., from jetstack.io, grafana.io)
2. `values.yaml` contains the configuration, namespaced under the dependency name
3. `overrides/values.yaml` exists for cluster-specific overrides (often empty)

Example `Chart.yaml`:
```yaml
apiVersion: v2
name: cert-manager
dependencies:
  - name: cert-manager
    version: ~v1.19
    repository: https://charts.jetstack.io
```

Example `values.yaml` structure (values nested under dependency name):
```yaml
cert-manager:
  crds:
    enabled: true
  replicaCount: 1
```

## Deployment

ArgoCD monitors this repository and automatically deploys changes when commits are pushed. Do not run `helm install/upgrade` commands manually.

## Common Commands

```bash
# Update chart dependencies (required after changing Chart.yaml)
helm dependency update <cluster>/<app>

# Template a chart (dry-run to verify changes)
helm template <release-name> <cluster>/<app> -f <cluster>/<app>/values.yaml
```

## Key Applications by Cluster

### nbg1-prod1
- **istio** - Service mesh (ambient mode with base, istiod, cni, ztunnel)
- **cert-manager** - Certificate management with CSI driver
- **external-secrets** - Secrets management
- **grafana** - Monitoring dashboards (exposed via Tailscale ingress)
- **loki** - Log aggregation
- **logging-agent** - Grafana Alloy for log collection
- **hcloud-csi** - Hetzner Cloud CSI driver
- **gateway-api** - Gateway API CRDs only (see README.md)

### raspberrypi1
- **kube-prometheus-stack** - Full Prometheus/Grafana monitoring stack
- **cert-manager**, **sealed-secrets**, **tailscale-operator**, **metrics-server**

## Istio Ambient Mode (nbg1-prod1)

Istio is deployed in **ambient mode** (v1.28.x) with these components:
- `base` - Base Istio CRDs
- `istiod` - Control plane (ambient profile)
- `cni` - CNI plugin for traffic redirection
- `ztunnel` - Zero-trust tunnel for transparent mTLS

### Istio Namespace
All Istio components deploy to the `istio` namespace.

### Adding Istio Policies
Custom Istio resources (AuthorizationPolicy, PeerAuthentication, etc.) should be added to:
```
nbg1-prod1/istio/templates/
```

### Service Mesh Topology

| Application | Namespace | Key Services | Notes |
|-------------|-----------|--------------|-------|
| Grafana | `grafana` | `grafana` | Exposed via Tailscale ingress |
| Loki | `loki` | `loki-gateway`, `loki-distributor`, `loki-ingester`, `loki-querier`, `loki-query-frontend`, `loki-compactor`, `loki-index-gateway`, `loki-chunks-cache`, `loki-results-cache` | Distributed mode with S3 storage |
| n8n | `n8n` | `n8n` | Exposed via Tailscale ingress |
| Logging Agent | `logging-agent` | `logging-agent-alloy` (DaemonSet) | Sends logs to Loki |

### Key Traffic Flows
- `logging-agent` → `loki-gateway.loki.svc.cluster.local` (log push via `/loki/api/v1/push`)
- `grafana` → `loki-gateway.loki.svc.cluster.local` (log queries)
- Tailscale Ingress → `grafana.grafana`, `n8n.n8n`

### Enabling Ambient for a Namespace
To add a namespace to the ambient mesh, label it:
```bash
kubectl label namespace <namespace> istio.io/dataplane-mode=ambient
```

## New App Onboarding (nbg1-prod1)

When deploying a new application to nbg1-prod1, follow this checklist:

### 1. Enable Prometheus Metrics

Metrics are scraped by `otel-metrics` (OpenTelemetry Collector) and stored in Mimir. The collector discovers pods using two methods:

#### Method A: Pod Annotations (Preferred)
Add these annotations to your pod spec in `values.yaml`:

```yaml
my-chart:
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"      # metrics port
    prometheus.io/path: "/metrics"  # optional, defaults to /metrics
    prometheus.io/scheme: "https"   # optional, for HTTPS endpoints
```

#### Method B: Port Name Convention
Name your metrics port with prefix `metrics` or `prometheus`:

```yaml
my-chart:
  service:
    ports:
      - name: metrics  # or prometheus-metrics, etc.
        port: 8080
```

#### Common Helm Chart Patterns

Different charts expose metrics configuration differently. Check `helm show values <chart>` for options:

```yaml
# Pattern 1: metrics.enabled flag
my-chart:
  metrics:
    enabled: true

# Pattern 2: serviceMonitor (ignored, but often enables metrics endpoint)
my-chart:
  serviceMonitor:
    enabled: true

# Pattern 3: Environment variable
my-chart:
  env:
    METRICS_ENABLED: "true"

# Pattern 4: Explicit podAnnotations
my-chart:
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
```

#### Examples from This Repo

| App | Method | Configuration |
|-----|--------|---------------|
| cert-manager | Built-in | `prometheus.enabled: true` |
| grafana | Annotations | `podAnnotations: prometheus.io/*` |
| loki | Built-in | Exposes on port 3100 by default |
| n8n | Env var | `config.n8n.metrics: true` |
| hcloud-csi | Flag | `metrics.enabled: true` |
| strimzi-kafka-operator | Annotations | `annotations: prometheus.io/*` |
| Kafka clusters | JMX Exporter | `metricsConfig` in Kafka CR |

### 2. Istio AuthorizationPolicy

If the app's namespace is in the Istio ambient mesh, you **must** allow `otel-metrics` to scrape it. Create an AuthorizationPolicy in `nbg1-prod1/istio/templates/`:

```yaml
# nbg1-prod1/istio/templates/authz-myapp.yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: myapp
  namespace: myapp
spec:
  action: ALLOW
  rules:
    # Allow Tailscale ingress (if exposed externally)
    - from:
        - source:
            namespaces: ["tailscale-operator"]
    # Allow otel-metrics to scrape Prometheus metrics
    - from:
        - source:
            namespaces: ["otel-metrics"]
```

For apps with multiple internal components, add a separate internal policy:

```yaml
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: myapp-internal
  namespace: myapp
spec:
  action: ALLOW
  rules:
    # Allow internal communication
    - from:
        - source:
            namespaces: ["myapp"]
    # Allow otel-metrics to scrape all components
    - from:
        - source:
            namespaces: ["otel-metrics"]
```

### 3. Verify Metrics Collection

After deploying, verify metrics are being collected:

1. Check the `up` metric in Grafana/Mimir for your app's targets
2. Query for app-specific metrics (e.g., `{app="myapp"}`)
3. If metrics are missing but `up=1`, check AuthorizationPolicy
4. If `up=0` or target missing, check pod annotations and port configuration

### Checklist Summary

- [ ] Enable metrics endpoint in Helm values
- [ ] Add pod annotations if not automatic
- [ ] Create AuthorizationPolicy allowing `otel-metrics` namespace
- [ ] Verify metrics appear in Mimir

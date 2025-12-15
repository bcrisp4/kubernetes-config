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

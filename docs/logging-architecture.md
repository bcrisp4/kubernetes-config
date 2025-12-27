# Logging Architecture: Two-Tier OpenTelemetry Collector with Istio Ambient Mesh

## Overview

The logging architecture uses a two-tier OpenTelemetry Collector setup to collect logs from both Talos host services and Kubernetes pods, while integrating with the Istio ambient mesh for secure communication to Loki.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Talos Host                                                                  │
│   kernel logs ──→ localhost:6050                                            │
│   service logs ──→ localhost:6051                                           │
└───────────────────────────┬─────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ otel-receiver (DaemonSet, hostNetwork: true, NOT in mesh)                   │
│                                                                             │
│   tcplog receiver (localhost:6050) ─┐                                       │
│   tcplog receiver (localhost:6051) ─┼→ OTLP exporter                        │
│                                     │         ↓                             │
│                                     │  otel-shipper:4317                    │
└─────────────────────────────────────┼───────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ otel-shipper (DaemonSet, hostNetwork: false, IN mesh)                       │
│                                                                             │
│   OTLP receiver (:4317) ←── receives Talos logs from otel-receiver          │
│          ↓                                                                  │
│   transform processor (extract service.name from talos-service)             │
│          ↓                                                                  │
│   filelog receiver (/var/log/pods) ←── pod logs                             │
│          ↓                                                                  │
│   otlphttp exporter ──→ loki-gateway.loki.svc/otlp (mTLS via mesh)          │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Components

### otel-receiver

| Property | Value |
|----------|-------|
| Type | DaemonSet |
| hostNetwork | `true` |
| Namespace | `otel-receiver` (not in mesh) |
| Purpose | Receive Talos host logs and forward to otel-shipper |
| Chart | `open-telemetry/opentelemetry-collector` |

**Responsibilities:**
- Listen on localhost:6050 for Talos kernel logs (JSON over TCP)
- Listen on localhost:6051 for Talos service logs (JSON over TCP)
- Parse JSON and add `log_type` attribute (talos-kernel or talos-service)
- Forward logs to otel-shipper via OTLP gRPC

**Configuration highlights:**
```yaml
receivers:
  tcplog/talos_service:
    listen_address: "127.0.0.1:6051"
    operators:
      - type: json_parser
        parse_from: body
        parse_to: attributes

  tcplog/talos_kernel:
    listen_address: "127.0.0.1:6050"
    operators:
      - type: json_parser
        parse_from: body
        parse_to: attributes

exporters:
  otlp:
    endpoint: "otel-shipper-opentelemetry-collector.otel-shipper.svc.cluster.local:4317"
    tls:
      insecure: true
```

### otel-shipper

| Property | Value |
|----------|-------|
| Type | DaemonSet |
| hostNetwork | `false` |
| Namespace | `otel-shipper` (in mesh with `istio.io/dataplane-mode=ambient`) |
| Purpose | Process all logs and ship to Loki |
| Chart | `open-telemetry/opentelemetry-collector` |

**Responsibilities:**
- Receive Talos logs from otel-receiver via OTLP
- Collect pod logs from `/var/log/pods` using the filelog preset
- Transform Talos logs to extract `service.name` from `talos-service` attribute
- Ship all logs to Loki via OTLP HTTP with mTLS (via mesh)

**Configuration highlights:**
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

presets:
  logsCollection:
    enabled: true
    includeCollectorLogs: false
    storeCheckpoints: true

processors:
  transform/talos:
    error_mode: ignore
    log_statements:
      - context: log
        statements:
          - set(severity_text, attributes["talos-level"]) where attributes["talos-level"] != nil
          - set(attributes["service.name"], Concat(["talos-", attributes["talos-service"]], "")) where attributes["talos-service"] != nil
          - set(attributes["service.name"], "talos-kernel") where resource.attributes["log_type"] == "talos-kernel" and attributes["service.name"] == nil

  groupbyattrs/talos:
    keys:
      - service.name

exporters:
  otlphttp/loki:
    endpoint: "http://loki-gateway.loki.svc.cluster.local/otlp"
    headers:
      X-Scope-OrgID: "prod"
```

## Namespace Configuration

| Namespace | Istio Ambient Label | Purpose |
|-----------|---------------------|---------|
| `otel-receiver` | None | Hosts otel-receiver (hostNetwork, outside mesh) |
| `otel-shipper` | `istio.io/dataplane-mode=ambient` | Hosts otel-shipper (in mesh) |

## Istio Authorization Policies

### otel-shipper Authorization

Since otel-receiver uses hostNetwork (outside the mesh), it cannot use mTLS. The otel-shipper requires:

1. **AuthorizationPolicy** allowing traffic from pod CIDR (10.244.0.0/16)
2. **PeerAuthentication** with PERMISSIVE mTLS mode

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: otel-shipper
  namespace: otel-shipper
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: opentelemetry-collector
  action: ALLOW
  rules:
    - from:
        - source:
            ipBlocks: ["10.244.0.0/16"]
---
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: otel-shipper
  namespace: otel-shipper
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: opentelemetry-collector
  mtls:
    mode: PERMISSIVE
```

### Loki Gateway Authorization

The Loki gateway only accepts traffic from mesh workloads:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: loki-gateway
  namespace: loki
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: gateway
  action: ALLOW
  rules:
    - from:
        - source:
            namespaces: ["otel-shipper"]
    - from:
        - source:
            namespaces: ["grafana"]
    - from:
        - source:
            namespaces: ["loki"]
```

## Resource Allocation

### otel-receiver

```yaml
resources:
  requests:
    cpu: 100m
    memory: 32Mi
  limits:
    memory: 64Mi
```

### otel-shipper

```yaml
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    memory: 256Mi
```

## Why Two Tiers?

The two-tier architecture solves a fundamental conflict with Istio ambient mesh:

1. **hostNetwork requirement**: Talos pushes logs to localhost ports on each node. A pod must use `hostNetwork: true` to bind to localhost.

2. **Mesh incompatibility**: Pods with `hostNetwork: true` don't work properly with ztunnel traffic interception. They cannot participate in the mesh.

3. **Security requirement**: We want mTLS for traffic to Loki, which requires the sender to be in the mesh.

**Solution**: Split into two components:
- **otel-receiver**: Uses hostNetwork to receive Talos logs, forwards to otel-shipper
- **otel-shipper**: Runs in pod network (in mesh), ships to Loki with mTLS

## Log Flow

1. **Talos host** pushes logs to localhost:6050 (kernel) and localhost:6051 (services)
2. **otel-receiver** receives logs via TCP, parses JSON, adds metadata, forwards via OTLP
3. **otel-shipper** receives from otel-receiver, collects pod logs, transforms, batches
4. **otel-shipper** ships to Loki gateway via OTLP HTTP (through mesh with mTLS)
5. **Loki** ingests and stores logs in S3

## Loki Integration

Loki v3 supports native OTLP ingestion via the `/otlp` endpoint, eliminating the need for the deprecated `lokiexporter`. The otel-shipper sends logs directly using the standard `otlphttp` exporter.

Labels are controlled via:
- Resource attributes (become Loki labels automatically)
- `loki.attribute.labels` hint for promoting log attributes to labels

## Resilience Configuration

Both collectors are configured with retry and queue settings to handle downstream outages gracefully:

```yaml
retry_on_failure:
  enabled: true
  initial_interval: 5s      # Initial backoff
  max_interval: 300s        # Max 5 min between retries
  max_elapsed_time: 600s    # Drop data after 10 min of retries
  randomization_factor: 0.5 # ±50% jitter to prevent thundering herd

sending_queue:
  enabled: true
  num_consumers: 10         # Parallel export workers
  queue_size: 5000          # Buffer capacity
```

**Key behaviors**:
- **Jitter**: Randomizes retry timing to prevent all collectors from retrying simultaneously after an outage
- **Bounded retry**: Drops data after 10 minutes to prevent memory exhaustion
- **Queue buffer**: Absorbs transient failures without blocking the pipeline

See [post-mortem: 2025-12-27 Mimir Kafka Outage](post-mortems/2025-12-27-mimir-kafka-outage.md) for background on why these settings were added.

## Monitoring

Both collectors expose Prometheus metrics on port 8888 with annotations for scraping by otel-metrics:

```yaml
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8888"
  prometheus.io/path: "/metrics"
```

## Files

| Path | Description |
|------|-------------|
| `nbg1-prod1/otel-receiver/` | otel-receiver Helm chart |
| `nbg1-prod1/otel-shipper/` | otel-shipper Helm chart |
| `nbg1-prod1/istio/templates/authz-otel-shipper.yaml` | Authorization policy for otel-shipper |
| `nbg1-prod1/istio/templates/authz-loki.yaml` | Authorization policy for Loki |

# Plan: Backup Prometheus for Meta-Monitoring

**Status**: Planned
**Related**: [2025-12-27 Mimir Kafka Outage Post-Mortem](../post-mortems/2025-12-27-mimir-kafka-outage.md)

## Overview

Deploy a standalone Prometheus instance to monitor Mimir, Kafka, and critical Kubernetes components independently. This ensures alerting works even when Mimir is down.

## Files to Create

### 1. `nbg1-prod1/prometheus-meta/Chart.yaml`

```yaml
apiVersion: v2
name: prometheus-meta
description: Standalone Prometheus for meta-monitoring
version: 0.1.0
dependencies:
  - name: prometheus
    version: ~26.0
    repository: https://prometheus-community.github.io/helm-charts
```

### 2. `nbg1-prod1/prometheus-meta/templates/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prometheus-meta
  labels:
    istio.io/dataplane-mode: ambient
```

### 3. `nbg1-prod1/prometheus-meta/values.yaml`

```yaml
prometheus:
  # Disable sub-charts - already deployed separately
  kube-state-metrics:
    enabled: false
  prometheus-node-exporter:
    enabled: false
  prometheus-pushgateway:
    enabled: false

  server:
    # Enable metrics scraping for self-monitoring
    podAnnotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "9090"

    # Short retention for meta-monitoring
    retention: "48h"
    retentionSize: "5GB"

    persistentVolume:
      enabled: true
      size: 10Gi

    resources:
      requests:
        cpu: 100m
        memory: 512Mi
      limits:
        memory: 1Gi

    # Tailscale ingress
    ingress:
      enabled: true
      ingressClassName: tailscale
      hosts:
        - prometheus-meta
      paths:
        - /
      tls:
        - secretName: prometheus-meta-tls
          hosts:
            - prometheus-meta

  alertmanager:
    enabled: true
    persistence:
      enabled: true
      size: 1Gi
    resources:
      requests:
        cpu: 25m
        memory: 64Mi
      limits:
        memory: 128Mi

  serverFiles:
    alerting_rules.yml:
      groups:
        - name: mimir-kafka-critical
          rules:
            - alert: KafkaDiskUsageHigh
              expr: |
                kubelet_volume_stats_used_bytes{namespace="mimir",persistentvolumeclaim=~"data-.*-mimir-kafka-.*"}
                / kubelet_volume_stats_capacity_bytes > 0.7
              for: 10m
              labels:
                severity: warning
              annotations:
                summary: "Kafka disk usage above 70%"
                description: "PVC {{ $labels.persistentvolumeclaim }} is at {{ $value | humanizePercentage }} capacity"

            - alert: KafkaDiskUsageCritical
              expr: |
                kubelet_volume_stats_used_bytes{namespace="mimir",persistentvolumeclaim=~"data-.*-mimir-kafka-.*"}
                / kubelet_volume_stats_capacity_bytes > 0.85
              for: 5m
              labels:
                severity: critical
              annotations:
                summary: "Kafka disk usage above 85%"
                description: "PVC {{ $labels.persistentvolumeclaim }} is at {{ $value | humanizePercentage }} capacity - URGENT"

            - alert: KafkaBrokerDown
              expr: up{job="mimir/kafka"} == 0
              for: 5m
              labels:
                severity: critical
              annotations:
                summary: "Kafka broker is down"
                description: "Kafka broker {{ $labels.pod }} has been down for 5+ minutes"

        - name: mimir-critical
          rules:
            - alert: MimirComponentDown
              expr: up{namespace="mimir", app=~"mimir-.*"} == 0
              for: 5m
              labels:
                severity: critical
              annotations:
                summary: "Mimir component is down"
                description: "{{ $labels.pod }} has been down for 5+ minutes"

            - alert: MimirDistributorHighMemory
              expr: |
                container_memory_working_set_bytes{namespace="mimir", container="mimir", pod=~".*distributor.*"}
                / container_spec_memory_limit_bytes > 0.8
              for: 5m
              labels:
                severity: warning
              annotations:
                summary: "Mimir distributor memory usage above 80%"
                description: "{{ $labels.pod }} is at {{ $value | humanizePercentage }} memory limit"

            - alert: MimirIngesterHighMemory
              expr: |
                container_memory_working_set_bytes{namespace="mimir", container="mimir", pod=~".*ingester.*"}
                / container_spec_memory_limit_bytes > 0.8
              for: 5m
              labels:
                severity: warning
              annotations:
                summary: "Mimir ingester memory usage above 80%"
                description: "{{ $labels.pod }} is at {{ $value | humanizePercentage }} memory limit"

        - name: kubernetes-critical
          rules:
            - alert: EtcdMembersDown
              expr: etcd_server_has_leader == 0
              for: 1m
              labels:
                severity: critical
              annotations:
                summary: "etcd member has no leader"
                description: "etcd member {{ $labels.instance }} reports no leader"

            - alert: KubeAPIServerDown
              expr: up{job="kube-apiserver"} == 0
              for: 5m
              labels:
                severity: critical
              annotations:
                summary: "Kubernetes API server is down"
                description: "API server is not responding to scrapes"

            - alert: NodeNotReady
              expr: kube_node_status_condition{condition="Ready",status="true"} == 0
              for: 10m
              labels:
                severity: critical
              annotations:
                summary: "Node is not ready"
                description: "Node {{ $labels.node }} has been NotReady for 10+ minutes"

    prometheus.yml:
      global:
        scrape_interval: 30s
        evaluation_interval: 30s

      alerting:
        alertmanagers:
          - static_configs:
              - targets:
                  - localhost:9093

      rule_files:
        - /etc/config/alerting_rules.yml

      scrape_configs:
        # Prometheus self-monitoring
        - job_name: prometheus
          static_configs:
            - targets:
                - localhost:9090

        # Mimir components (all expose on port 3100)
        - job_name: mimir
          kubernetes_sd_configs:
            - role: pod
              namespaces:
                names:
                  - mimir
          relabel_configs:
            - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
              action: keep
              regex: mimir-distributed
            - source_labels: [__meta_kubernetes_pod_container_port_number]
              action: keep
              regex: "3100"
            - source_labels: [__meta_kubernetes_namespace]
              target_label: namespace
            - source_labels: [__meta_kubernetes_pod_name]
              target_label: pod
            - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_component]
              target_label: component
            - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
              target_label: app

        # Kafka brokers (JMX exporter on port 9404)
        - job_name: mimir/kafka
          kubernetes_sd_configs:
            - role: pod
              namespaces:
                names:
                  - mimir
          relabel_configs:
            - source_labels: [__meta_kubernetes_pod_label_strimzi_io_cluster]
              action: keep
              regex: mimir-kafka
            - source_labels: [__meta_kubernetes_pod_container_port_number]
              action: keep
              regex: "9404"
            - source_labels: [__meta_kubernetes_namespace]
              target_label: namespace
            - source_labels: [__meta_kubernetes_pod_name]
              target_label: pod

        # Kubelet metrics
        - job_name: kubelet
          scheme: https
          kubernetes_sd_configs:
            - role: node
          bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
          tls_config:
            ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
            insecure_skip_verify: true
          relabel_configs:
            - source_labels: [__meta_kubernetes_node_address_InternalIP]
              target_label: __address__
              regex: (.+)
              replacement: $1:10250
            - source_labels: [__meta_kubernetes_node_name]
              target_label: node
            - source_labels: [__meta_kubernetes_node_name]
              target_label: instance
            - target_label: job
              replacement: kubelet

        # cAdvisor (container metrics)
        - job_name: kubelet-cadvisor
          scheme: https
          metrics_path: /metrics/cadvisor
          kubernetes_sd_configs:
            - role: node
          bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
          tls_config:
            ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
            insecure_skip_verify: true
          relabel_configs:
            - source_labels: [__meta_kubernetes_node_address_InternalIP]
              target_label: __address__
              regex: (.+)
              replacement: $1:10250
            - source_labels: [__meta_kubernetes_node_name]
              target_label: node
            - source_labels: [__meta_kubernetes_node_name]
              target_label: instance
            - target_label: job
              replacement: kubelet

        # etcd (control plane nodes only)
        - job_name: etcd
          scheme: http
          kubernetes_sd_configs:
            - role: node
          relabel_configs:
            - source_labels: [__meta_kubernetes_node_labelpresent_node_role_kubernetes_io_control_plane]
              action: keep
              regex: "true"
            - source_labels: [__meta_kubernetes_node_address_InternalIP]
              target_label: __address__
              regex: (.+)
              replacement: $1:2381
            - source_labels: [__meta_kubernetes_node_name]
              target_label: node
            - source_labels: [__meta_kubernetes_node_name]
              target_label: instance
            - target_label: job
              replacement: etcd

        # kube-apiserver
        - job_name: kube-apiserver
          scheme: https
          kubernetes_sd_configs:
            - role: endpoints
              namespaces:
                names:
                  - default
          bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
          tls_config:
            ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
            insecure_skip_verify: true
          relabel_configs:
            - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
              action: keep
              regex: kubernetes;https
            - target_label: job
              replacement: kube-apiserver

        # kube-state-metrics (for PVC and node metrics)
        - job_name: kube-state-metrics
          kubernetes_sd_configs:
            - role: pod
              namespaces:
                names:
                  - kube-state-metrics
          relabel_configs:
            - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
              action: keep
              regex: kube-state-metrics
            - source_labels: [__meta_kubernetes_pod_container_port_number]
              action: keep
              regex: "8080"
            - source_labels: [__meta_kubernetes_namespace]
              target_label: namespace
            - source_labels: [__meta_kubernetes_pod_name]
              target_label: pod

  rbac:
    create: true
  serviceAccounts:
    server:
      create: true
```

### 4. `nbg1-prod1/prometheus-meta/overrides/values.yaml`

```yaml
# Empty - for cluster-specific overrides
```

## Files to Modify

### 5. `nbg1-prod1/istio/templates/authz-mimir.yaml`

Add to the `mimir-internal` AuthorizationPolicy rules:

```yaml
    # Allow prometheus-meta to scrape metrics
    - from:
        - source:
            namespaces: ["prometheus-meta"]
```

### 6. `nbg1-prod1/istio/templates/authz-prometheus-meta.yaml` (new file)

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: prometheus-meta
  namespace: prometheus-meta
spec:
  action: ALLOW
  rules:
    # Allow Tailscale ingress for UI access
    - from:
        - source:
            namespaces: ["tailscale-operator"]
    # Allow internal traffic (prometheus -> alertmanager)
    - from:
        - source:
            namespaces: ["prometheus-meta"]
```

## Implementation Steps

1. Create directory structure:
   ```bash
   mkdir -p nbg1-prod1/prometheus-meta/{templates,overrides}
   ```

2. Create the files listed above

3. Update Istio AuthorizationPolicy for mimir namespace

4. Add new AuthorizationPolicy for prometheus-meta namespace

5. Update helm dependencies:
   ```bash
   helm dependency update nbg1-prod1/prometheus-meta
   ```

6. Verify with dry-run:
   ```bash
   helm template prometheus-meta nbg1-prod1/prometheus-meta -f nbg1-prod1/prometheus-meta/values.yaml
   ```

7. Commit and push - ArgoCD will deploy automatically

8. Update post-mortem to mark task complete

## Resource Summary

| Component | CPU Request | Memory Request | Memory Limit | Storage |
|-----------|-------------|----------------|--------------|---------|
| Prometheus | 100m | 512Mi | 1Gi | 10Gi |
| Alertmanager | 25m | 64Mi | 128Mi | 1Gi |
| **Total** | 125m | 576Mi | ~1.1Gi | 11Gi |

## Alert Rules Summary

| Alert | Trigger | Severity |
|-------|---------|----------|
| KafkaDiskUsageHigh | >70% for 10m | warning |
| KafkaDiskUsageCritical | >85% for 5m | critical |
| KafkaBrokerDown | up=0 for 5m | critical |
| MimirComponentDown | up=0 for 5m | critical |
| MimirDistributorHighMemory | >80% for 5m | warning |
| MimirIngesterHighMemory | >80% for 5m | warning |
| EtcdMembersDown | no leader for 1m | critical |
| KubeAPIServerDown | up=0 for 5m | critical |
| NodeNotReady | NotReady for 10m | critical |

## Future Enhancements

- Add Slack/PagerDuty notification integration
- Dead man's switch (Watchdog alert)
- More granular Mimir alerts (ingestion rate, query latency)
- OTEL collector health alerts

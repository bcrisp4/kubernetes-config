# Incident Post-Mortem: Mimir Kafka Disk Full Outage

**Date:** 2025-12-27
**Duration:** ~2 hours
**Severity:** High (metrics ingestion fully impacted)
**Author:** Claude Code / Ben

## Summary

Mimir's Kafka cluster ran out of disk space, causing a cascading failure that stopped all metrics ingestion. When Kafka recovered, a thundering herd of OTEL collectors overwhelmed the distributors, causing OOM kills and prolonged recovery.

## Impact

- Complete loss of metrics ingestion for ~2 hours
- Some metrics data lost during the outage window
- Partial data loss during recovery due to rate limiting and dropped samples

## Timeline (UTC)

| Time | Event |
|------|-------|
| ~20:00 | Kafka brokers begin running low on disk space |
| ~20:57 | All 3 Kafka brokers enter CrashLoopBackOff with "No space left on device" |
| 01:27 | Issue identified - Kafka PVCs at 100% (10Gi each) |
| 01:58 | Fix deployed: PVC increased to 50Gi, retention reduced 24h → 6h |
| 02:00 | Kafka brokers recover and become Ready |
| 02:00 | Distributors begin crashing with OOMKilled (thundering herd) |
| 02:03 | Distributor memory limit increased 512Mi → 1Gi |
| 02:05 | Ingestion rate limits increased 100k → 500k samples/s |
| 02:08 | Full recovery - metrics flowing at ~183k samples/s |

## Root Cause Analysis

### Primary Cause: Kafka Disk Exhaustion

The Strimzi Kafka cluster was provisioned with 10Gi PVCs and 24-hour log retention. With Mimir's ingest traffic, this was insufficient:

- 12 partitions × 3 replicas = high replication overhead
- 24-hour retention accumulated more data than disk capacity
- No alerting on Kafka disk usage

### Secondary Cause: Thundering Herd

When Kafka recovered, all 27 OTEL collector pods (across otel-metrics, otel-receiver, otel-shipper namespaces) simultaneously attempted to flush their buffered metrics:

1. OTEL collectors use exponential backoff with retry queues
2. During outage, collectors accumulated backlogged data
3. When Mimir became available, all collectors retried within seconds
4. Combined load exceeded distributor capacity and rate limits
5. Distributors OOMKilled trying to process the spike
6. OOM caused more retries, creating a feedback loop

### Contributing Factors

1. **Distributor memory limits too low** (512Mi) for burst traffic
2. **Ingestion rate limits** (100k/s) appropriate for steady-state but not recovery
3. **No circuit breaker** between OTEL collectors and Mimir
4. **OTEL retry behavior** lacks sufficient jitter to prevent synchronized retries

## Resolution

### Immediate Fixes

1. **Kafka storage** (`kafka-cluster.yaml`):
   ```yaml
   # Increased PVC size
   size: 50Gi  # was 10Gi

   # Reduced retention
   log.retention.hours: 6  # was 24
   log.segment.bytes: 134217728  # 128MB segments for faster cleanup
   ```

2. **Distributor resources** (`values.yaml`):
   ```yaml
   distributor:
     resources:
       requests:
         memory: 512Mi  # was 256Mi
       limits:
         memory: 1Gi    # was 512Mi
   ```

3. **Ingestion rate limits** (`values.yaml`):
   ```yaml
   prod:
     ingestion_rate: 500000      # was 100000
     ingestion_burst_size: 1000000  # was 200000
   ```

## Follow-up Actions

### P1 - Prevent Thundering Herd

- [x] **Add jitter to OTEL collector retry intervals**

  Configured all OTEL exporters (otel-metrics, otel-receiver, otel-shipper) with randomized retry delays:

  ```yaml
  exporters:
    otlphttp/mimir:
      retry_on_failure:
        enabled: true
        initial_interval: 5s
        max_interval: 300s
        max_elapsed_time: 600s  # Drop data after 10 min of retries
        randomization_factor: 0.5  # Add 50% jitter
      sending_queue:
        enabled: true
        num_consumers: 10
        queue_size: 5000
  ```

- [ ] **Consider rate limiting at OTEL collector level**

  Add a rate limiter processor to prevent any single collector from overwhelming Mimir:

  ```yaml
  processors:
    rate_limiter:
      check_interval: 1s
      # Limit each collector to a fraction of total capacity
      # With 27 collectors and 500k limit: ~15k per collector
      limit_mps: 15000
  ```

### P2 - Improve Observability

- [ ] **Deploy backup Prometheus for meta-monitoring**

  When Mimir is down, alerts don't fire because alerting depends on Mimir. Deploy a lightweight standalone Prometheus instance with:
  - Short retention (24-48h) to minimize resource usage
  - Scrape only critical targets: Mimir components, Kafka, key Kubernetes components
  - Alertmanager integration for critical infrastructure alerts
  - Does NOT send data to Mimir (independent monitoring path)

  This ensures we get alerted about Mimir/Kafka issues even when Mimir itself is broken.

- [ ] **Add Kafka disk usage alerts** (to backup Prometheus)

  Create alerting rule for Kafka PVC usage > 70%:

  ```yaml
  - alert: KafkaDiskUsageHigh
    expr: |
      kubelet_volume_stats_used_bytes{namespace="mimir",persistentvolumeclaim=~"data-.*-mimir-kafka-.*"}
      / kubelet_volume_stats_capacity_bytes > 0.7
    for: 10m
    labels:
      severity: warning
  ```

- [ ] **Add distributor memory usage alerts**

  Alert when distributors approach memory limits

### P3 - Architecture Improvements

- [ ] **Evaluate Kafka auto-scaling**

  Consider Strimzi Kafka auto-scaling for storage based on usage

- [ ] **Review ingestion rate limits**

  Determine if 500k/s should be the new baseline or revert to 100k/s after backlog clears

- [ ] **Consider circuit breaker pattern**

  Evaluate implementing a circuit breaker (e.g., via Istio) between OTEL and Mimir to prevent cascading failures

- [ ] **OTEL collector horizontal pod autoscaler**

  Consider HPA for OTEL collectors based on queue depth

## Lessons Learned

1. **Disk capacity planning is critical for Kafka** - Always provision for at least 2x expected retention
2. **Burst capacity matters** - Steady-state limits are insufficient for recovery scenarios
3. **Distributed retry without jitter causes thundering herd** - Always add randomization
4. **Memory limits should account for queued data** - Distributors need headroom for burst traffic
5. **Alerting gaps** - We had no visibility into Kafka disk usage before it became critical

## References

- [Mimir Ingest Storage Configuration](https://grafana.com/docs/mimir/latest/configure/configure-kafka-backend/)
- [OTEL Collector Retry Configuration](https://github.com/open-telemetry/opentelemetry-collector/blob/main/exporter/exporterhelper/README.md)
- [Strimzi Kafka Storage](https://strimzi.io/docs/operators/latest/configuring.html#con-common-configuration-storage-reference)

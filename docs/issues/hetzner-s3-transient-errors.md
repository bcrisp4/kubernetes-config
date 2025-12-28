# Hetzner Object Storage - Transient S3 Errors

**Status:** Open - Monitoring
**Date:** 2025-12-27
**Affected Service:** Mimir (compactor, ruler)
**Endpoint:** `nbg1.your-objectstorage.com`
**Bucket:** `bc4-mimir-nbg1-prod1`

## Summary

Mimir components are experiencing intermittent S3 errors when accessing Hetzner Object Storage. The errors are transient - the same operations succeed on retry. This is causing compaction delays and occasional ruler failures.

## Error Types Observed

### 1. "Bucket does not exist" (transient)

```
level=warn msg="failed to download block" err="get object: GetObjectInput.Bucket: bucket \"bc4-mimir-nbg1-prod1\" does not exist"
```

This error is misleading - the bucket definitely exists. The error resolves on its own without intervention.

### 2. "Unexpected EOF"

```
level=warn msg="failed to download block" err="unexpected EOF"
```

Occurs during large block downloads, suggesting connection resets or timeouts.

## Timeline

| Time | Event |
|------|-------|
| 2025-12-27 ~01:00 | First observed during compactor recovery after Kafka outage |
| 2025-12-27 ~02:30 | Added HTTP timeout configuration to mitigate slow responses |
| 2025-12-27 ~03:00 | Switched to new bucket name, errors persist |
| Ongoing | Errors continue intermittently |

## Configuration Applied

Added HTTP timeout settings to all S3 storage blocks in Mimir to handle slow responses:

```yaml
s3:
  bucket_name: bc4-mimir-nbg1-prod1
  endpoint: nbg1.your-objectstorage.com
  http:
    idle_conn_timeout: 2m
    response_header_timeout: 5m
    max_idle_connections: 100
    max_idle_connections_per_host: 100
```

## Observations

1. **Loki works fine** - Same S3 endpoint and credentials work reliably for Loki chunks storage
2. **Transient nature** - Errors resolve without intervention, operations succeed on retry
3. **No pattern identified** - Errors don't correlate with time of day or load
4. **Bucket name doesn't matter** - Errors occurred with both old (`bc4-nbg1-prod1-mimir`) and new (`bc4-mimir-nbg1-prod1`) bucket names

## Differences: Mimir vs Loki S3 Usage

| Aspect | Mimir | Loki |
|--------|-------|------|
| Object sizes | Large blocks (100MB+) | Smaller chunks |
| Access pattern | Compactor downloads many blocks | More streaming writes |
| Concurrency | High during compaction | Distributed across ingesters |

## Potential Causes

1. **Rate limiting** - Hetzner may be throttling high-concurrency requests
2. **DNS resolution** - Transient DNS issues returning wrong endpoints
3. **Load balancer issues** - Backend S3 nodes becoming temporarily unavailable
4. **Network path issues** - Intermittent connectivity between Hetzner compute and storage

## Next Steps

1. **Contact Hetzner Support** - Report the issue with:
   - Bucket name: `bc4-mimir-nbg1-prod1`
   - Endpoint: `nbg1.your-objectstorage.com`
   - Error types: "bucket does not exist" and "unexpected EOF"
   - Timestamps of recent occurrences
   - Note that errors are transient and affect high-concurrency workloads

2. **Request from Hetzner:**
   - Server-side logs for the affected bucket during error windows
   - Any known issues with NBG1 object storage
   - Rate limits or throttling policies that might apply
   - Recommended settings for high-throughput workloads

3. **Potential mitigations to discuss:**
   - Dedicated/reserved capacity
   - Different endpoint or region
   - Recommended retry/backoff settings

## Workarounds in Place

- HTTP timeouts configured for slow responses
- Mimir components retry failed operations automatically
- Compaction eventually succeeds despite transient failures

## Related

- [Mimir Kafka Outage Post-Mortem](../post-mortems/2025-12-27-mimir-kafka-outage.md) - S3 issues surfaced during recovery

# Hetzner Object Storage - Transient S3 Errors

**Status:** Resolved - Migrated to FSN1 region
**Date:** 2025-12-27
**Resolution Date:** 2025-12-28
**Affected Service:** Mimir (compactor, ruler)
**Original Endpoint:** `nbg1.your-objectstorage.com`
**Original Bucket:** `bc4-mimir-nbg1-prod1`
**New Endpoint:** `fsn1.your-objectstorage.com`
**New Bucket:** `bc4-mimir-nbg1-prod1-fsn1`

## Summary

Mimir components were experiencing intermittent S3 errors when accessing Hetzner Object Storage in the NBG1 region. The errors were transient - the same operations would succeed on retry. This caused compaction delays and occasional ruler failures.

**Resolution:** Migrated to FSN1 region which does not exhibit these issues. The NBG1 region appears to have infrastructure-level issues with high-concurrency S3 workloads.

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

### 3. "GatewayTimeout"

```
An error occurred (GatewayTimeout) when calling the UploadPartCopy operation (reached max retries: 2): The server did not respond in time.
```

Occurs during multipart upload operations, particularly for larger objects.

### 4. "404 Not Found" on HeadObject

```
An error occurred (404) when calling the HeadObject operation: Not Found
```

Object exists but HeadObject returns 404 transiently.

## Timeline

| Time | Event |
|------|-------|
| 2025-12-27 ~01:00 | First observed during compactor recovery after Kafka outage |
| 2025-12-27 ~02:30 | Added HTTP timeout configuration to mitigate slow responses |
| 2025-12-27 ~03:00 | Switched to new bucket name, errors persist |
| 2025-12-28 | Stability testing: NBG1 showed 10% failure rate (3/30 operations) |
| 2025-12-28 | Created new bucket in FSN1 region |
| 2025-12-28 | FSN1 stability test: 0% failure rate (0/30 operations) |
| 2025-12-28 | Migrated Mimir to FSN1 bucket, old NBG1 bucket retained as backup |

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

## Reproduction with AWS CLI

The issue is reproducible with standard AWS CLI tools, not just Mimir. During a bucket-to-bucket copy operation using `aws s3 sync`, transient errors occurred on random objects while other objects in the same batch succeeded:

```bash
# Command used
aws s3 sync s3://bc4-nbg1-prod1-mimir/blocks/prod/ s3://bc4-mimir-nbg1-prod1/blocks/prod/ \
  --endpoint-url https://nbg1.your-objectstorage.com

# Example errors during sync (interleaved with successful copies)
copy failed: s3://bc4-nbg1-prod1-mimir/blocks/prod/01KDF25JK9VQY7ZY4HN9GYD08H/meta.json to s3://bc4-mimir-nbg1-prod1/blocks/prod/01KDF25JK9VQY7ZY4HN9GYD08H/meta.json An error occurred (NoSuchBucket) when calling the CopyObject operation: The specified bucket does not exist.

copy failed: s3://bc4-nbg1-prod1-mimir/blocks/prod/01KDFV1M5KC3N4749P3BJ3CVVA/chunks/000001 to s3://bc4-mimir-nbg1-prod1/blocks/prod/01KDFV1M5KC3N4749P3BJ3CVVA/chunks/000001 An error occurred (GatewayTimeout) when calling the UploadPartCopy operation (reached max retries: 2): The server did not respond in time.

copy failed: s3://bc4-nbg1-prod1-mimir/blocks/prod/01KDF5DX49N0EJX98HN7SDM13N/chunks/000001 to s3://bc4-mimir-nbg1-prod1/blocks/prod/01KDF5DX49N0EJX98HN7SDM13N/chunks/000001 An error occurred (404) when calling the HeadObject operation: Not Found
```

**Key observations from CLI testing:**
- Same request succeeds on retry (running `aws s3 sync` again copies the failed files)
- Errors occur mid-batch - some objects succeed, others fail, in same operation
- Multiple error types: `NoSuchBucket`, `GatewayTimeout`, `404 Not Found`
- Large multipart uploads more likely to fail (`UploadPartCopy` operations)

## Observations

1. **Loki works fine** - Same S3 endpoint and credentials work reliably for Loki chunks storage
2. **Transient nature** - Errors resolve without intervention, operations succeed on retry
3. **No pattern identified** - Errors don't correlate with time of day or load
4. **Bucket name doesn't matter** - Errors occurred with both old (`bc4-nbg1-prod1-mimir`) and new (`bc4-mimir-nbg1-prod1`) bucket names
5. **Reproducible with AWS CLI** - Not specific to Mimir's S3 client implementation

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

## Original Next Steps (No Longer Required)

These steps were planned before the FSN1 migration resolved the issue:

1. ~~**Contact Hetzner Support** - Report the issue~~
2. ~~**Request server-side logs from Hetzner**~~
3. ~~**Discuss potential mitigations**~~

The issue was resolved by migrating to FSN1 region.

## Resolution

Migrated to FSN1 region which does not exhibit the same transient errors.

### Stability Testing Results

Before migration, we ran identical tests against both regions:

**Test methodology:** 20 ListObjectsV2 + 10 HeadObject operations per bucket

| Region | Bucket | Errors | Failure Rate |
|--------|--------|--------|--------------|
| NBG1 | `bc4-mimir-nbg1-prod1` | 3/30 | 10% |
| FSN1 | `bc4-mimir-nbg1-prod1-fsn1` | 0/30 | 0% |

### Cross-Datacenter Testing

To rule out inter-datacenter networking issues, we tested from nodes in both NBG1 and FSN1 datacenters:

| Source Node DC | Target S3 Region | Errors | Failure Rate |
|----------------|------------------|--------|--------------|
| NBG1 | NBG1 S3 | 6/30 | 20% |
| NBG1 | FSN1 S3 | 0/30 | 0% |
| FSN1 | NBG1 S3 | 3/30 | 10% |
| FSN1 | FSN1 S3 | 0/30 | 0% |

**Conclusion:** Both NBG1 and FSN1 nodes experience errors when accessing NBG1 S3, while both work perfectly with FSN1 S3. This confirms the issue is with NBG1 Object Storage infrastructure, not inter-datacenter networking.

### Migration Steps

1. Deleted 2 incomplete blocks from NBG1 bucket (missing index files from prior failed copies)
2. Copied all 102 objects from NBG1 to FSN1 bucket
3. Required retry logic due to NBG1 transient failures during copy
4. Verified all objects present in FSN1
5. Updated Mimir configuration to use FSN1 endpoint and bucket
6. NBG1 bucket retained as backup

### Post-Migration Configuration

```yaml
s3:
  bucket_name: bc4-mimir-nbg1-prod1-fsn1
  endpoint: fsn1.your-objectstorage.com
  # HTTP timeout settings retained for safety
  http:
    idle_conn_timeout: 2m
    response_header_timeout: 5m
    max_idle_connections: 100
    max_idle_connections_per_host: 100
```

### Conclusion

The transient S3 errors appear to be specific to the NBG1 region's object storage infrastructure. FSN1 region does not exhibit the same issues. No contact with Hetzner support was needed.

**Note:** All traffic originates from Kubernetes nodes running inside Hetzner Cloud (spread across NBG1 and FSN1 datacenters), so this is entirely internal Hetzner infrastructure - not external network issues.

## Previous Workarounds (No Longer Required)

- HTTP timeouts configured for slow responses
- Mimir components retry failed operations automatically
- Compaction eventually succeeds despite transient failures

## Hetzner Support Request

The following can be sent to Hetzner support if needed:

---

**Subject:** Transient S3 errors on NBG1 Object Storage - NoSuchBucket, GatewayTimeout, 404

**Issue:**
We're experiencing intermittent S3 API errors on NBG1 Object Storage. Operations fail transiently but succeed on immediate retry. This affects our metrics storage system (Grafana Mimir) which makes high-concurrency S3 requests.

**Affected bucket:** `bc4-mimir-nbg1-prod1`
**Endpoint:** `nbg1.your-objectstorage.com`
**First observed:** 2025-12-27

**Error types observed:**
- `NoSuchBucket` - "The specified bucket does not exist" (bucket exists, error is transient)
- `GatewayTimeout` - "The server did not respond in time" (on UploadPartCopy)
- `404 Not Found` - on HeadObject for objects that exist
- `unexpected EOF` - during large object downloads

**Reproducible with AWS CLI:**
```
aws s3 sync s3://source/ s3://bc4-mimir-nbg1-prod1/dest/ --endpoint-url https://nbg1.your-objectstorage.com
# Random objects fail mid-batch, succeed on retry
```

**Testing confirms NBG1-specific issue:**

| Client Location | Target Endpoint | Errors (30 ops) |
|-----------------|-----------------|-----------------|
| NBG1 Cloud VM | NBG1 S3 | 6 (20%) |
| FSN1 Cloud VM | NBG1 S3 | 3 (10%) |
| NBG1 Cloud VM | FSN1 S3 | 0 (0%) |
| FSN1 Cloud VM | FSN1 S3 | 0 (0%) |

All traffic originates from Hetzner Cloud VMs (internal). FSN1 Object Storage works perfectly from both datacenters. We have migrated to FSN1 as a workaround.

**Questions:**
1. Are there known issues with NBG1 Object Storage?
2. Is there rate limiting or throttling that could cause these transient errors?
3. Can you check server-side logs for bucket `bc4-mimir-nbg1-prod1` around 2025-12-27?

---

## Related

- [Mimir Kafka Outage Post-Mortem](../post-mortems/2025-12-27-mimir-kafka-outage.md) - S3 issues surfaced during recovery

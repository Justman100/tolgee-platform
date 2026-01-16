# Phase 1 Implementation Log

Reference documents:
- [PLAN.md](./PLAN.md)
- [PHASE_1.md](./PHASE_1.md)

**Note:** This log was created retroactively after implementation completed. Some details may be incomplete.

---

## Step 0: Create and Verify Tracing Stack

### Context
Creating a local Docker Compose environment with OpenTelemetry tracing infrastructure to enable end-to-end testing of trace collection before production deployment.

---

### Step 0.1: Create Directory Structure

**Command:**
```bash
mkdir -p /Users/gabriel.shanahan/projects/tolgee/tolgee-platform/docker/tracing/grafana
```

**Timestamp:** 2026-01-16T16:48:00Z (approximate)

**Result:** SUCCESS

**Rollback:** `rm -rf docker/tracing`

---

### Step 0.2: Create Configuration Files

**Files created:**
- `docker/tracing/tempo-config.yaml`
- `docker/tracing/otel-collector-config.yaml`
- `docker/tracing/loki-config.yaml`
- `docker/tracing/promtail-config.yaml`
- `docker/tracing/grafana/datasources.yaml`
- `docker/docker-compose.tracing.yaml`

**Timestamp:** 2026-01-16T16:48:30Z (approximate)

**Result:** SUCCESS

**Rollback:** `rm -rf docker/tracing docker/docker-compose.tracing.yaml`

---

### Step 0.3: First Start Attempt - Translator Auth Failure

**Command:**
```bash
cd docker && docker compose -f docker-compose.tracing.yaml up -d
```

**Timestamp:** 2026-01-16T16:49:00Z (approximate)

**Result:** FAILURE
```
translator Error Head "https://ghcr.io/v2/tolgee/translator/manifests/v1.9.0": unauthorized
```

**Resolution:** Removed translator service from docker-compose.tracing.yaml (not needed for tracing infrastructure testing)

**Rollback:** N/A (failure, no state change)

---

### Step 0.4: Second Start Attempt - Redis Image Not Found

**Command:**
```bash
cd docker && docker compose -f docker-compose.tracing.yaml up -d
```

**Timestamp:** 2026-01-16T16:50:00Z (approximate)

**Result:** FAILURE
```
redis Error manifest for bitnami/redis:7.0.5 not found: manifest unknown
```

**Resolution:** Changed redis image from `bitnami/redis:7.0.5` to `redis:7-alpine` with command `["redis-server", "--requirepass", "tolgee"]`

**Rollback:** N/A (failure, no state change)

---

### Step 0.5: Third Start Attempt - Network Timeouts

**Command:**
```bash
cd docker && docker compose -f docker-compose.tracing.yaml pull
```

**Timestamp:** 2026-01-16T16:51:00Z (approximate)

**Result:** FAILURE - Multiple network timeouts to Docker Hub

**Resolution:** Pull images individually

**Rollback:** N/A (failure, no state change)

---

### Step 0.6: Pull Images Individually

**Commands:**
```bash
docker pull grafana/tempo:2.6.1
docker pull grafana/loki:2.9.1
docker pull grafana/promtail:2.9.1
docker pull grafana/grafana:11.4.0
docker pull otel/opentelemetry-collector-contrib:0.115.0  # FAILED - version doesn't exist
docker pull otel/opentelemetry-collector-contrib:0.96.0   # SUCCESS
docker pull redis:7-alpine
```

**Timestamp:** 2026-01-16T16:51:30Z (approximate)

**Result:** PARTIAL SUCCESS
- OTel Collector version 0.115.0 does not exist
- Changed to version 0.96.0

**Correction Applied:**
Updated `docker/docker-compose.tracing.yaml`:
- Changed `otel/opentelemetry-collector-contrib:0.115.0` to `otel/opentelemetry-collector-contrib:0.96.0`

Updated `docs/implementation-log/2026-01-16-add-tracing/PLAN.md`:
- Changed OTel Collector version in component table

Updated `docs/implementation-log/2026-01-16-add-tracing/PHASE_1.md`:
- Changed OTel Collector version in docker-compose example

**Rollback:** `docker rmi grafana/tempo:2.6.1 grafana/loki:2.9.1 grafana/promtail:2.9.1 grafana/grafana:11.4.0 otel/opentelemetry-collector-contrib:0.96.0 redis:7-alpine`

---

### Step 0.7: Start Stack Successfully

**Command:**
```bash
cd docker && docker compose -f docker-compose.tracing.yaml up -d
```

**Timestamp:** 2026-01-16T16:51:36Z

**Result:** SUCCESS
```
Container tolgee-tracing-postgres-1    Started
Container tolgee-tracing-redis-1       Started
Container tolgee-tracing-loki-1        Started
Container tolgee-tracing-tempo-1       Started
Container tolgee-tracing-promtail-1    Started
Container tolgee-tracing-otel-collector-1  Started
Container tolgee-tracing-grafana-1     Started
```

**Rollback:** `cd docker && docker compose -f docker-compose.tracing.yaml down -v`

---

### Step 0.8: Fix OTel Collector Healthcheck

**Issue:** OTel Collector showed "health: starting" indefinitely because healthcheck used `wget` which is not available in the image.

**Verification:**
```bash
curl -s http://localhost:13133/
# {"status":"Server available","upSince":"2026-01-16T16:51:36.87458155Z","uptime":"47.598664313s"}
```

**Correction Applied:**
Updated `docker/docker-compose.tracing.yaml`:
```yaml
# Changed from:
healthcheck:
  test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:13133/"]

# Changed to:
healthcheck:
  test: ["CMD-SHELL", "curl -f http://localhost:13133/ || exit 1"]
```

Updated `docs/implementation-log/2026-01-16-add-tracing/PHASE_1.md` with same fix.

**Rollback:** Revert the healthcheck change in docker-compose.tracing.yaml

---

### Step 0.9: Send Test Trace (Old Timestamps)

**Command:**
```bash
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {"attributes": [{"key": "service.name", "value": {"stringValue": "test-service"}}]},
      "scopeSpans": [{
        "spans": [{
          "traceId": "5B8EFFF798038103D269B633813FC60C",
          "spanId": "EEE19B7EC3C1B174",
          "name": "test-span",
          "kind": 1,
          "startTimeUnixNano": "1544712660000000000",
          "endTimeUnixNano": "1544712661000000000",
          "status": {"code": 2}
        }]
      }]
    }]
  }'
```

**Timestamp:** 2026-01-16T16:52:30Z (approximate)

**Result:** SUCCESS - Trace accepted, but timestamps from 2018 so not visible in "last 1 hour" time range in Grafana.

**Rollback:** N/A (test data only)

---

### Step 0.10: Verify Trace in Tempo API

**Command:**
```bash
curl -s "http://localhost:3200/api/traces/5b8efff798038103d269b633813fc60c"
```

**Result:** SUCCESS
```json
{"batches":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"test-service"}}]},"scopeSpans":[{"scope":{},"spans":[{"traceId":"W47/95gDgQPSabYzgT/GDA==","spanId":"7uGbfsPBsXQ=","name":"test-span","kind":"SPAN_KIND_INTERNAL","startTimeUnixNano":"1544712660000000000","endTimeUnixNano":"1544712661000000000","status":{"code":"STATUS_CODE_ERROR"}}]}]}]}
```

**Rollback:** N/A (read-only verification)

---

### Step 0.11: Send Test Trace (Current Timestamps)

**Command:**
```bash
CURRENT_NANO=$(python3 -c "import time; print(int(time.time() * 1e9))")
END_NANO=$((CURRENT_NANO + 1000000000))
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d "{
    \"resourceSpans\": [{
      \"resource\": {\"attributes\": [{\"key\": \"service.name\", \"value\": {\"stringValue\": \"test-service\"}}]},
      \"scopeSpans\": [{
        \"spans\": [{
          \"traceId\": \"D64B28CBBD7B791E313D6856F76FA33C\",
          \"spanId\": \"...\",
          \"name\": \"test-span-now\",
          \"kind\": 1,
          \"startTimeUnixNano\": \"$CURRENT_NANO\",
          \"endTimeUnixNano\": \"$END_NANO\",
          \"status\": {\"code\": 2}
        }]
      }]
    }]
  }"
```

**Timestamp:** 2026-01-16T17:58:12Z

**Result:** SUCCESS

**Rollback:** N/A (test data only)

---

### Step 0.12: Verify Trace in Grafana UI

**Actions:**
1. Opened http://localhost:3000 in browser
2. Navigated to Explore (menu → Explore)
3. Selected Tempo datasource (pre-configured)
4. Entered TraceQL query: `{}`
5. Clicked "Run query"
6. Trace appeared in results table:
   - Trace ID: d64b28cbbd7b791e313d6856f76fa33c
   - Service: test-service
   - Name: test-span-now
   - Duration: 1s
7. Clicked on trace to view details
8. Confirmed trace timeline visualization working
9. Confirmed trace-to-logs link to Loki visible

**Timestamp:** 2026-01-16T17:59:00Z (approximate)

**Result:** SUCCESS - Full end-to-end verification complete

**Screenshot saved:** `docs/implementation-log/2026-01-16-add-tracing/grafana-trace-view.png`

**Rollback:** N/A (read-only verification)

---

## Step 0 Summary

| Resource | Status | Rollback |
|----------|--------|----------|
| `docker/tracing/` directory | CREATED | `rm -rf docker/tracing/` |
| `docker/docker-compose.tracing.yaml` | CREATED | `rm docker/docker-compose.tracing.yaml` |
| Docker images pulled | PULLED | `docker rmi grafana/tempo:2.6.1 grafana/loki:2.9.1 grafana/promtail:2.9.1 grafana/grafana:11.4.0 otel/opentelemetry-collector-contrib:0.96.0 redis:7-alpine` |
| Docker containers/volumes | RUNNING | `cd docker && docker compose -f docker-compose.tracing.yaml down -v` |
| PLAN.md | MODIFIED | Revert version changes |
| PHASE_1.md | MODIFIED | Revert version changes |

---

## Corrections Summary

| Original Value | Corrected Value | Reason |
|----------------|-----------------|--------|
| `ghcr.io/tolgee/translator:v1.9.0` | (removed) | Requires ghcr.io authentication |
| `bitnami/redis:7.0.5` | `redis:7-alpine` | Image not found |
| `otel/opentelemetry-collector-contrib:0.115.0` | `otel/opentelemetry-collector-contrib:0.96.0` | Version doesn't exist |
| OTel Collector healthcheck using `wget` | Using `curl` | wget not available in image |

---

## Full Rollback

To completely undo all changes from this phase:

```bash
# Stop and remove containers and volumes
cd /Users/gabriel.shanahan/projects/tolgee/tolgee-platform/docker
docker compose -f docker-compose.tracing.yaml down -v

# Remove created files
rm -rf docker/tracing/
rm -f docker/docker-compose.tracing.yaml

# Remove documentation (optional)
rm -rf docs/implementation-log/2026-01-16-add-tracing/

# Remove pulled images (optional)
docker rmi grafana/tempo:2.6.1 grafana/loki:2.9.1 grafana/promtail:2.9.1 grafana/grafana:11.4.0 otel/opentelemetry-collector-contrib:0.96.0 redis:7-alpine
```

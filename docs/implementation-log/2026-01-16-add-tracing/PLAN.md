# Local Tracing Environment Setup

## Overview

Create a Docker Compose environment with all infrastructure components for OpenTelemetry tracing. This enables end-to-end testing of trace collection, OTel Collector tail-based sampling, and log correlation before deploying any changes to production.

This is Phase 1 of the tracing implementation plan.

## Goals

1. Self-contained local tracing stack that mirrors production architecture
2. Test trace collection end-to-end (app → OTel Collector → Tempo → Grafana)
3. Validate OTel Collector tail-based sampling (same config as production)
4. Test trace-to-logs correlation via Loki

## Non-Goals

- Production deployment configuration (separate scope)
- Application instrumentation (Phase 2)
- Billing service instrumentation (separate scope)

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Local Development Stack                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                   │
│  │  Tolgee App │────>│OTel Collector────>│    Tempo    │                   │
│  │  (traces)   │     │(tail sampling)    │  (storage)  │                   │
│  └──────┬──────┘     └─────────────┘     └──────┬──────┘                   │
│         │                                       │                           │
│         │  ┌────────────────────────────────────┘                           │
│         │  │                                                                │
│         ▼  ▼                                                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │ PostgreSQL  │  │   Redis     │  │ Translator  │  │   Grafana   │       │
│  │   (DB)      │  │ (Sentinel)  │  │   (ML)      │  │   (UI)      │       │
│  │   :5432     │  │ :6379/:26379│  │   :4000     │  │   :3000     │       │
│  └─────────────┘  └─────────────┘  └─────────────┘  └──────┬──────┘       │
│                                                             │               │
│                                                             ▼               │
│                                                      ┌─────────────┐       │
│                                                      │    Loki     │       │
│                                                      │   (logs)    │       │
│                                                      │   :3100     │       │
│                                                      └─────────────┘       │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Component versions (must match production):**

| Component | Docker Image | Port | Notes |
|-----------|--------------|------|-------|
| Tempo | grafana/tempo:2.6.1 | 3200 | Monolithic mode locally |
| OTel Collector | otel/opentelemetry-collector-contrib:0.96.0 | 4317, 4318 | Same tail sampling config |
| Loki | grafana/loki:2.9.1 | 3100 | Log storage |
| Promtail | grafana/promtail:2.9.1 | - | Log collection |
| Grafana | grafana/grafana:11.4.0 | 3000 | Visualization |
| Redis | redis:7-alpine | 6379 | Standard Redis |
| PostgreSQL | postgres:15 | 5432 | Matches testing env version |

## Phases

### Phase 1: Local Tracing Stack

Create complete Docker Compose environment with all configuration files and services.

**Files to create:**
- `docker/tracing/tempo-config.yaml` - Tempo in monolithic mode
- `docker/tracing/otel-collector-config.yaml` - OTel Collector with tail sampling
- `docker/tracing/loki-config.yaml` - Loki log storage
- `docker/tracing/promtail-config.yaml` - Promtail log collection
- `docker/tracing/grafana/datasources.yaml` - Pre-configured data sources
- `docker/docker-compose.tracing.yaml` - Complete tracing stack

**Key decisions:**
- Tempo in monolithic mode (simpler than distributed, same OTLP interface)
- OTel Collector tail sampling: errors + latency >2s + 10% probabilistic
- Grafana pre-configured with Tempo/Loki correlation
- Named volumes for persistence (postgres-data, redis-data, tempo-data, loki-data, grafana-data)
- All infrastructure Tolgee needs included (PostgreSQL, Redis)

## Rollback Strategy

```bash
# Remove all created files
rm -rf docker/tracing/
rm docker/docker-compose.tracing.yaml

# Stop and remove containers/volumes if started
docker compose -f docker/docker-compose.tracing.yaml down -v
```

## Open Questions

None - all decisions made in the high-level plan.

---

## Verification

After implementation:

```bash
# 1. Start the stack
cd tolgee-platform/docker
docker compose -f docker-compose.tracing.yaml up -d

# 2. Check all services running
docker compose -f docker-compose.tracing.yaml ps

# 3. Verify Grafana datasources work
open http://localhost:3000
# Navigate to Connections → Data sources
# Test Tempo and Loki connections

# 4. Send test trace
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

# 5. Verify trace appears in Grafana Explore → Tempo
```

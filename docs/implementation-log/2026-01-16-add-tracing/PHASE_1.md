# Phase 1: Local Tracing Stack

## Overview

| Step | Description | Test Method |
|------|-------------|-------------|
| 0 | Create and verify tracing stack | All services healthy, test trace visible in Grafana |

---

## Step 0: Create and Verify Tracing Stack

### Goal
Create all configuration files, start the stack, and verify end-to-end trace flow.

### Prerequisites
- Working directory is `tolgee-platform`
- Docker running

### Instructions

Create directory structure:
```bash
mkdir -p docker/tracing/grafana
```

Create `docker/tracing/tempo-config.yaml`:
```yaml
stream_over_http_enabled: true
server:
  http_listen_port: 3200
  log_level: info

query_frontend:
  search:
    duration_slo: 5s
    throughput_bytes_slo: 1.073741824e+09
  trace_by_id:
    duration_slo: 5s

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

ingester:
  max_block_duration: 5m

compactor:
  compaction:
    block_retention: 48h

storage:
  trace:
    backend: local
    wal:
      path: /var/tempo/wal
    local:
      path: /var/tempo/blocks

overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics]
```

Create `docker/tracing/otel-collector-config.yaml`:
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    expected_new_traces_per_sec: 100
    policies:
      - name: errors-policy
        type: status_code
        status_code:
          status_codes:
            - ERROR
      - name: latency-policy
        type: latency
        latency:
          threshold_ms: 2000
      - name: probabilistic-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

  debug:
    verbosity: detailed

extensions:
  health_check:
    endpoint: 0.0.0.0:13133

service:
  extensions: [health_check]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling, batch]
      exporters: [otlp/tempo, debug]
```

Create `docker/tracing/loki-config.yaml`:
```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: info

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 24

analytics:
  reporting_enabled: false
```

Create `docker/tracing/promtail-config.yaml`:
```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: containers
    static_configs:
      - targets:
          - localhost
        labels:
          job: containerlogs
          __path__: /var/lib/docker/containers/*/*log

    pipeline_stages:
      - json:
          expressions:
            output: log
            stream: stream
            attrs:
      - json:
          expressions:
            tag:
          source: attrs
      - regex:
          expression: ^(?P<container_name>[^|]+)\|(?P<image_name>[^$]+)$
          source: tag
      - timestamp:
          source: time
          format: RFC3339Nano
      - labels:
          stream:
          container_name:
          image_name:
      - output:
          source: output
      - regex:
          expression: '(?:trace_id|traceId|traceid)["\s:=]+(?P<trace_id>[a-fA-F0-9]{32})'
          source: output
      - labels:
          trace_id:
```

Create `docker/tracing/grafana/datasources.yaml`:
```yaml
apiVersion: 1

datasources:
  - name: Tempo
    type: tempo
    access: proxy
    orgId: 1
    url: http://tempo:3200
    basicAuth: false
    isDefault: true
    version: 1
    editable: false
    apiVersion: 1
    uid: tempo
    jsonData:
      httpMethod: GET
      tracesToLogs:
        datasourceUid: loki
        spanStartTimeShift: '-1h'
        spanEndTimeShift: '1h'
        tags: ['container_name', 'service.name']
        filterByTraceID: true
        filterBySpanID: false
        mapTagNamesEnabled: true
        mappedTags:
          - key: service.name
            value: service_name
      nodeGraph:
        enabled: true
      search:
        hide: false
      lokiSearch:
        datasourceUid: loki

  - name: Loki
    type: loki
    access: proxy
    orgId: 1
    url: http://loki:3100
    basicAuth: false
    isDefault: false
    version: 1
    editable: false
    apiVersion: 1
    uid: loki
    jsonData:
      derivedFields:
        - datasourceUid: tempo
          matcherRegex: '(?:trace_id|traceId|traceid)["\s:=]+([a-fA-F0-9]{32})'
          name: TraceID
          url: '$${__value.raw}'
          urlDisplayLabel: 'View Trace'
```

Create `docker/docker-compose.tracing.yaml`:
```yaml
name: tolgee-tracing

services:
  # Tracing Infrastructure
  tempo:
    image: grafana/tempo:2.6.1
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./tracing/tempo-config.yaml:/etc/tempo.yaml:ro
      - tempo-data:/var/tempo
    ports:
      - "3200:3200"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3200/ready"]
      interval: 10s
      timeout: 5s
      retries: 5

  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.96.0
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./tracing/otel-collector-config.yaml:/etc/otel-collector-config.yaml:ro
    ports:
      - "4317:4317"
      - "4318:4318"
      - "13133:13133"
    depends_on:
      tempo:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:13133/ || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5

  loki:
    image: grafana/loki:2.9.1
    command: ["-config.file=/etc/loki/local-config.yaml"]
    volumes:
      - ./tracing/loki-config.yaml:/etc/loki/local-config.yaml:ro
      - loki-data:/loki
    ports:
      - "3100:3100"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3100/ready"]
      interval: 10s
      timeout: 5s
      retries: 5

  promtail:
    image: grafana/promtail:2.9.1
    command: ["-config.file=/etc/promtail/config.yaml"]
    volumes:
      - ./tracing/promtail-config.yaml:/etc/promtail/config.yaml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      loki:
        condition: service_healthy

  grafana:
    image: grafana/grafana:11.4.0
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_AUTH_DISABLE_LOGIN_FORM=false
    volumes:
      - ./tracing/grafana/datasources.yaml:/etc/grafana/provisioning/datasources/datasources.yaml:ro
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    depends_on:
      tempo:
        condition: service_healthy
      loki:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/api/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Tolgee Dependencies
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: tolgee
      POSTGRES_PASSWORD: tolgee
      POSTGRES_DB: tolgee
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U tolgee"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--requirepass", "tolgee"]
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "tolgee", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  tempo-data:
  loki-data:
  grafana-data:
  postgres-data:
  redis-data:
```

Start the stack:
```bash
cd docker
docker compose -f docker-compose.tracing.yaml up -d
```

Wait for all services to be healthy:
```bash
docker compose -f docker-compose.tracing.yaml ps
```

Send a test trace:
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

Verify in Grafana:
1. Open http://localhost:3000
2. Go to Explore → Select Tempo
3. Search for trace ID: `5B8EFFF798038103D269B633813FC60C`

### Testing
- All containers show "healthy" in `docker compose ps`
- Test trace appears in Grafana

### Checklist
- [ ] All services healthy
- [ ] Test trace visible in Grafana

### Notes
```
Space for runtime notes during implementation.
```

---

## Rollback

```bash
# Stop and remove containers and volumes
cd docker
docker compose -f docker-compose.tracing.yaml down -v

# Remove all created files
rm -rf docker/tracing/
rm -f docker/docker-compose.tracing.yaml
```

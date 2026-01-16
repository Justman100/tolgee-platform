# Backend OpenTelemetry Instrumentation

## Overview

Add OpenTelemetry auto-instrumentation to the tolgee-platform backend using the [OpenTelemetry Java agent](https://opentelemetry.io/docs/zero-code/java/agent/). This enables distributed tracing for HTTP requests, database queries, Redis operations, and 100+ other libraries, with traces exported to the local tracing stack (from Phase 1) or production infrastructure.

## Goals

1. Add OpenTelemetry Java agent dependency to the project
2. Configure bootRun to attach the agent via `-javaagent` flag
3. Auto-instrument HTTP, database, Redis, and other operations (no code changes required)
4. Tracing disabled by default, enabled via `OTEL_JAVAAGENT_ENABLED=true`

## Non-Goals

- Production deployment configuration (separate scope)
- Custom span creation/manual instrumentation (can be added later)
- Log-to-trace correlation (handled by infrastructure)

## Prerequisites

- Phase 1 complete: Local tracing stack running (`docker/docker-compose.tracing.yaml`)

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    tolgee-platform backend                       │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │         OpenTelemetry Java Agent (-javaagent)              │  │
│  │         Bytecode instrumentation at JVM startup            │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │  HTTP       │───>│  Service    │───>│  Repository │         │
│  │  Controller │    │  Layer      │    │  Layer      │         │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘         │
│         │                  │                  │                 │
│         │    Auto-instrumented by Java Agent                   │
│         │    ────────────────────────────────                   │
│         ▼                  ▼                  ▼                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              OTLP Exporter (HTTP, port 4318)             │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │                                   │
└─────────────────────────────┼───────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  OTel Collector │
                    │  (localhost:4318)│
                    └─────────────────┘
```

## Files to Modify

- `settings.gradle` - Add OpenTelemetry Java agent to version catalog
- `backend/app/build.gradle` - Add otelAgent configuration and bootRun setup

**No application code changes required.** The Java agent instruments bytecode at JVM startup.

## Configuration

Configuration is done via environment variables (standard OpenTelemetry configuration):

| Variable | Purpose | Default |
|----------|---------|---------|
| `OTEL_JAVAAGENT_ENABLED` | Enable agent in bootRun (custom) | `false` |
| `OTEL_SERVICE_NAME` | Service name in traces | `unknown_service:java` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Collector endpoint | `http://localhost:4318` |
| `OTEL_RESOURCE_ATTRIBUTES` | Additional resource attributes | - |
| `OTEL_TRACES_EXPORTER` | Exporter type (`otlp`, `none`) | `otlp` |
| `OTEL_JAVAAGENT_DEBUG` | Enable debug logging | `false` |

See [full configuration reference](https://opentelemetry.io/docs/zero-code/java/agent/configuration/).

## Phases

### Phase 1: Add Java Agent Configuration

Add OpenTelemetry Java agent dependency and configure bootRun to use it.

**Key decisions:**
- Use [OpenTelemetry Java agent](https://opentelemetry.io/docs/zero-code/java/agent/) for bytecode instrumentation
- Agent version 2.24.0
- OTLP HTTP exporter (not gRPC) for simpler setup
- Enabled via `OTEL_JAVAAGENT_ENABLED=true` environment variable

### Phase 2: Documentation

Plan and implement documentation for the tracing feature following the [Documentation as a Separate Phase](../CLAUDE.md#documentation-as-a-separate-phase) process.

**Deliverables:**
1. `DOCS_PLAN.md` - Documentation hierarchy plan (created in this implementation-log folder)
2. Code documentation - Constants with KDoc, links to external docs, entry point documentation

**Key documentation needs:**
- OpenTelemetry concepts for readers unfamiliar with tracing
- Environment variable configuration (what they mean, why these values)
- How this integrates with the local tracing stack (Phase 1 of cross-repo plan)
- Entry point for newcomers to understand the tracing setup

## Rollback Strategy

```bash
# Revert modified files
git checkout -- settings.gradle
git checkout -- backend/app/build.gradle

# Or disable at runtime: OTEL_JAVAAGENT_ENABLED=false (or simply don't set it)
```

## Verification

1. Start local tracing stack (Phase 1)
2. Start Tolgee with tracing enabled:
   ```bash
   OTEL_JAVAAGENT_ENABLED=true \
   OTEL_SERVICE_NAME=tolgee-platform \
   OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 \
   OTEL_RESOURCE_ATTRIBUTES=deployment.environment=local \
   ./gradlew server-app:bootRun --args='--spring.profiles.active=dev'
   ```
3. Look for agent startup log: `[otel.javaagent] opentelemetry-javaagent - version: 2.24.0`
4. Make API request: `curl http://localhost:8080/v2/public/initial-data`
5. Verify trace in Grafana: Explore → Tempo → `{service.name="tolgee-platform"}`

## Open Questions

None - approach determined by cross-repo plan.

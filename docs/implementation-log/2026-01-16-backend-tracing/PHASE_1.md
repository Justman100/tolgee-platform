# Phase 1: Add Dependencies and Configuration

## Overview

| Step | Description | Test Method |
|------|-------------|-------------|
| 0 | Add OpenTelemetry instrumentation | App starts with tracing enabled, traces visible in Grafana |

**Note:** This phase uses the [OpenTelemetry Java agent](https://opentelemetry.io/docs/zero-code/java/agent/) for bytecode instrumentation. The agent provides comprehensive auto-instrumentation for 100+ libraries (HTTP, JDBC, Redis, Kafka, gRPC, etc.) without any code changes. Phase 2 will add comprehensive documentation following the [Code Documentation Principle](../../../CLAUDE.md#code-documentation-principle).

---

## Handling Unexpected Situations

**This is critical.** During execution, if you encounter something unexpected that requires a non-read operation not pre-approved in this document, you MUST:

1. **STOP** - Do not proceed with the unexpected action
2. **ALERT** - Inform the user what happened and what action you believe is needed
3. **WAIT** - Let the user decide whether to proceed

**Examples of unexpected situations requiring pause:**
- A required dependency doesn't exist or has a different version than specified
- You discover the plan needs a step that wasn't anticipated
- A different approach is needed to solve a problem
- A database migration is needed that wasn't in the plan
- An unexpected error occurs that requires manual intervention
- Significant changes to parts of the codebase that are not covered by this document
- If at any point you've realized that you've significantly deviated from the plan (regardless of whether the deviation is correct and necessary or not)

**Exception:** Read-only operations (checking versions, listing files, reading code) to gather information are fine without approval.

---

## Step 0: Add OpenTelemetry Java Agent

### Goal
Add OpenTelemetry auto-instrumentation to the backend using the Java agent, so HTTP requests, database queries, Redis operations, and more automatically generate traces.

### Prerequisites
- Phase 1 of cross-repo plan complete (tracing stack running via `docker/docker-compose.tracing.yaml`)
- Working directory is `tolgee-platform`

### Instructions

**1. Add OpenTelemetry Java agent dependency to `settings.gradle`**

In the `libs` version catalog block (around line 77, after `jakartaMail`), add:

```groovy
// OpenTelemetry Java Agent for bytecode instrumentation
// See: https://opentelemetry.io/docs/zero-code/java/agent/
library('opentelemetryJavaagent', 'io.opentelemetry.javaagent', 'opentelemetry-javaagent').version('2.24.0')
```

**2. Configure the agent in `backend/app/build.gradle`**

Add a custom configuration for the agent and configure `bootRun` to use it.

First, add the configuration block near the top of the file (after the `plugins` block):

```groovy
/**
 * OpenTelemetry Java Agent configuration.
 * The agent is downloaded as a dependency and attached to the JVM via -javaagent flag.
 * See: https://opentelemetry.io/docs/zero-code/java/agent/
 */
configurations {
    otelAgent
}
```

Then add the dependency (around line 128, after the MISC section):

```groovy
/**
 * OPENTELEMETRY
 */
otelAgent libs.opentelemetryJavaagent
```

Finally, configure `bootRun` to use the agent. Find the existing `bootRun` configuration block and update it:

```groovy
bootRun {
    // ... existing configuration ...

    // OpenTelemetry Java Agent - disabled by default, enable via OTEL_JAVAAGENT_ENABLED=true
    // See: https://opentelemetry.io/docs/zero-code/java/agent/configuration/
    if (System.getenv("OTEL_JAVAAGENT_ENABLED") == "true") {
        jvmArgs "-javaagent:${configurations.otelAgent.singleFile.absolutePath}"
    }
}
```

**That's it.** The Java agent auto-instruments:
- HTTP requests (inbound via Spring MVC/WebFlux, outbound via RestTemplate, WebClient, Apache HttpClient, OkHttp, etc.)
- Database queries (JDBC, Hibernate, R2DBC)
- Redis (Jedis, Lettuce)
- Messaging (Kafka, RabbitMQ)
- gRPC, GraphQL, and [100+ more libraries](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/supported-libraries.md)

No code changes, no Spring configuration, no application.yaml changes required.

### Testing

1. Ensure tracing infrastructure is running:
```bash
cd docker && docker compose -f docker-compose.tracing.yaml ps
# All services should be healthy
```

2. Start Tolgee with tracing enabled:
```bash
OTEL_JAVAAGENT_ENABLED=true \
OTEL_SERVICE_NAME=tolgee-platform \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 \
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=local \
./gradlew server-app:bootRun --args='--spring.profiles.active=dev \
  --spring.datasource.url=jdbc:postgresql://localhost:5432/tolgee \
  --spring.datasource.username=tolgee \
  --spring.datasource.password=tolgee \
  --tolgee.postgres-autostart.enabled=false'
```

You should see agent startup logs:
```
[otel.javaagent] opentelemetry-javaagent - version: 2.24.0
```

3. Make a test request:
```bash
curl -X GET http://localhost:8080/v2/public/initial-data
```

4. Verify in Grafana (http://localhost:3000):
   - Go to Explore → Select Tempo
   - Query: `{service.name="tolgee-platform"}`
   - Should see traces with HTTP and database spans

### Environment Variables Reference

| Variable | Purpose | Default |
|----------|---------|---------|
| `OTEL_JAVAAGENT_ENABLED` | Enable agent in bootRun (custom) | `false` |
| `OTEL_SERVICE_NAME` | Service name in traces | `unknown_service:java` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Collector endpoint | `http://localhost:4318` |
| `OTEL_RESOURCE_ATTRIBUTES` | Additional resource attributes | - |
| `OTEL_TRACES_EXPORTER` | Exporter type (`otlp`, `none`) | `otlp` |
| `OTEL_JAVAAGENT_DEBUG` | Enable debug logging | `false` |

See [full configuration reference](https://opentelemetry.io/docs/zero-code/java/agent/configuration/).

### Checklist
- [x] Agent dependency added to settings.gradle
- [x] otelAgent configuration added to backend/app/build.gradle
- [x] bootRun configured with -javaagent flag
- [x] App compiles successfully (`./gradlew server-app:compileKotlin`)
- [x] App starts with tracing enabled (agent logs visible)
- [x] Traces visible in Grafana
- [x] **Log entry appended to PHASE_1_LOG.md**

### Notes
```
Space for runtime notes during implementation.
```

---

## Rollback

```bash
# Revert modified files
git checkout -- settings.gradle
git checkout -- backend/app/build.gradle
```

---

## References

- [OpenTelemetry Java Agent](https://opentelemetry.io/docs/zero-code/java/agent/) - Official documentation
- [Getting Started](https://opentelemetry.io/docs/zero-code/java/agent/getting-started/) - Quick start guide
- [Configuration](https://opentelemetry.io/docs/zero-code/java/agent/configuration/) - All environment variables and system properties
- [Supported Libraries](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/supported-libraries.md) - Full list of auto-instrumented libraries
- [GitHub Releases](https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases) - Agent downloads

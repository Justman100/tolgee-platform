# Phase 2: Documentation

## Overview

| Step | Description | Test Method |
|------|-------------|-------------|
| 0 | Create DOCS_PLAN.md with documentation hierarchy | File exists with complete plan |
| 0.5 | Enable log-trace correlation | Trace IDs appear in app logs, Grafana links work |
| 1 | Create Grafana Tracing Tutorial | Tutorial file exists at docker/tracing/GRAFANA_TUTORIAL.md |
| 2 | Add entry point documentation to docker-compose.tracing.yaml | Comments explain purpose, link to tutorial |
| 3 | Add inline documentation to configuration files | All config files have explanatory comments |
| 4 | Update DEVELOPMENT.md with tracing section | Tracing mentioned with links |
| 5 | Verify documentation hierarchy | Newcomer can follow the trail |

**Note:** This phase focuses on documentation following the [Documentation as a Separate Phase](../CLAUDE.md#documentation-as-a-separate-phase) process. The primary deliverable is a comprehensive Grafana tutorial for developers who have never worked with distributed tracing.

---

## Handling Unexpected Situations

**This is critical.** During execution, if you encounter something unexpected that requires a non-read operation not pre-approved in this document, you MUST:

1. **STOP** - Do not proceed with the unexpected action
2. **ALERT** - Inform the user what happened and what action you believe is needed
3. **WAIT** - Let the user decide whether to proceed

**Examples of unexpected situations requiring pause:**
- A file to be documented doesn't exist
- The documentation structure needs changes not anticipated here
- Screenshots cannot be captured due to infrastructure issues

**Exception:** Read-only operations (checking files, reading code) to gather information are fine without approval.

---

## Step 0: Create DOCS_PLAN.md

### Goal
Create a documentation hierarchy plan that maps out what needs to be documented and where readers should start.

### Prerequisites
- Phase 1 complete (tracing infrastructure working)
- Working directory is `tolgee-platform`

### Instructions

Create `docs/implementation-log/2026-01-16-backend-tracing/DOCS_PLAN.md` with the following content:

```markdown
# Documentation Plan: Backend Tracing

## Documentation Hierarchy

```
DEVELOPMENT.md (Entry Point)
    └── "Tracing" section
        └── docker/tracing/GRAFANA_TUTORIAL.md (Main Tutorial)
            ├── Part A: What is Tracing?
            ├── Part B: Navigating Grafana
            ├── Part C: Finding What You Need
            └── Part D: Real-World Use Cases

docker/docker-compose.tracing.yaml (Secondary Entry Point)
    └── Header comment with quick start + link to tutorial

docker/tracing/*.yaml (Configuration Files)
    └── Inline comments explaining each setting
```

## External Concepts to Document

| Concept | Where to Document | Approach |
|---------|-------------------|----------|
| Distributed tracing | GRAFANA_TUTORIAL.md Part A | 2-3 paragraphs explaining traces/spans |
| TraceQL | GRAFANA_TUTORIAL.md Part C | Examples with explanations |
| Tail sampling | GRAFANA_TUTORIAL.md Part A | Brief note about what gets sampled |
| OpenTelemetry | Link only | Reference to official docs |

## Target Audience

Developers who:
- Have never worked with distributed tracing
- Need to debug slow or failing requests
- Want to understand request flow in the codebase
```

### Testing
- File exists with complete hierarchy
- All documentation targets identified

### Checklist
- [ ] DOCS_PLAN.md created
- [ ] Entry point identified (DEVELOPMENT.md)
- [ ] All components to document listed
- [ ] External concepts catalogued
- [ ] **Log entry appended to PHASE_2_LOG.md**

---

## Step 0.5: Enable Log-Trace Correlation

### Goal
Configure the application to inject trace IDs into log output so that Grafana can link between traces and logs.

### Context
The Grafana datasources are already configured for log-trace correlation:
- Tempo has `tracesToLogs` pointing to Loki
- Loki has `derivedFields` to extract trace IDs from logs
- Promtail extracts `trace_id` from log content

**However**, the OpenTelemetry Java agent does NOT automatically inject trace IDs into logs. We need to configure MDC (Mapped Diagnostic Context) injection so that trace/span IDs appear in log output.

### Prerequisites
- Phase 1 complete (OpenTelemetry agent configured)
- Working directory is `tolgee-platform`

### Instructions

**Option A: Use OpenTelemetry Logback Appender (Recommended)**

The OTel Java agent can automatically inject trace context into Logback MDC. This requires adding the `otel.instrumentation.logback-mdc.enabled=true` configuration.

1. Add environment variable when starting the app:
   ```bash
   OTEL_INSTRUMENTATION_LOGBACK_MDC_ENABLED=true
   ```

2. Update logback configuration to include trace ID in log pattern. Find the logging configuration file (e.g., `logback-spring.xml` or similar) and update the pattern to include:
   ```xml
   %X{trace_id} %X{span_id}
   ```

**Option B: Alternative - Configure via System Property**

Add to JVM args:
```
-Dotel.instrumentation.logback-mdc.enabled=true
```

### Investigation Needed

Before implementation, verify:
1. Where is the Logback configuration file in Tolgee? (Search for `logback*.xml`)
2. What is the current log pattern?
3. Does Tolgee use Logback or another logging framework?

**Note:** This step may require creating a PHASE_1.5.md if code changes are needed beyond configuration. If significant code changes are required, pause and discuss with user.

### Testing

1. Start app with tracing + MDC enabled:
   ```bash
   OTEL_JAVAAGENT_ENABLED=true \
   OTEL_INSTRUMENTATION_LOGBACK_MDC_ENABLED=true \
   OTEL_SERVICE_NAME=tolgee-platform \
   OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 \
   ./gradlew server-app:bootRun --args='--spring.profiles.active=dev'
   ```

2. Make a request and check logs contain trace ID:
   ```bash
   curl http://localhost:8080/v2/public/initial-data
   # Check application logs for trace_id/span_id fields
   ```

3. In Grafana:
   - Go to Explore → Loki
   - Query: `{container_name=~".*"} |= "trace_id"`
   - Verify "View Trace" link appears on log lines
   - Click link and verify it opens the corresponding trace in Tempo

4. Reverse direction:
   - Go to Explore → Tempo
   - Find a trace
   - Click "Logs for this span" button
   - Verify associated logs appear

### Checklist
- [ ] Logging framework identified (Logback/Log4j2/other)
- [ ] MDC injection enabled via environment variable
- [ ] Log pattern updated to include trace_id (if needed)
- [ ] Trace IDs visible in application logs
- [ ] Grafana Loki → Tempo link works
- [ ] Grafana Tempo → Loki link works
- [ ] Screenshot captured: `15-log-trace-correlation.png`
- [ ] **Log entry appended to PHASE_2_LOG.md**

---

## Step 1: Create Grafana Tracing Tutorial

### Goal
Create a comprehensive tutorial for developers unfamiliar with tracing, covering both "how to use" and "what to use it for."

### Prerequisites
- Tracing stack running (`docker compose -f docker-compose.tracing.yaml up -d`)
- At least one trace visible in Grafana (from Phase 1 testing)

### Instructions

**1. Create the tutorial file**

Create `docker/tracing/GRAFANA_TUTORIAL.md` with the structure below.

**2. Take screenshots**

Create directory `docker/tracing/images/` and capture screenshots as you write each section.

### Tutorial Content

```markdown
# Grafana Tracing Tutorial

> **For:** Developers who have never worked with distributed tracing before
>
> **Goal:** Learn how to use Grafana to find and analyze traces, and understand when tracing helps you debug problems

## Table of Contents

1. [What is Tracing and Why Does It Matter?](#part-a-what-is-tracing-and-why-does-it-matter)
2. [Navigating Grafana](#part-b-navigating-grafana)
3. [Finding What You Need](#part-c-finding-what-you-need)
4. [Real-World Use Cases](#part-d-real-world-use-cases)
5. [Span Metrics and Time-Series Analysis](#part-e-span-metrics-and-time-series-analysis)

---

## Part A: What is Tracing and Why Does It Matter?

### The Problem Tracing Solves

When a user reports "the API is slow" or "this request failed," you need to answer:
- Where did the time go?
- Which database query was slow?
- What external service call failed?
- What was the sequence of operations?

**Without tracing:** You grep through logs, correlate timestamps manually, and guess.

**With tracing:** You see the complete journey of a request through your system, with timing for each step.

### Mental Model: Traces and Spans

**Trace:** A single request's journey through the system, from start to finish.
- Has a unique **Trace ID** (32-character hex string like `5B8EFFF798038103D269B633813FC60C`)
- Contains one or more **spans**

**Span:** A single operation within a trace (e.g., "handle HTTP request", "execute SQL query").
- Has a name describing the operation
- Has a duration (start time + end time)
- Can have **child spans** (operations that happened during this span)
- Contains **attributes** (metadata like `http.status_code=200`, `db.statement=SELECT...`)

**Visual representation:**
```
Trace: 5B8EFFF798038103D269B633813FC60C
├── HTTP GET /v2/projects/123 ──────────────────────── [250ms]
│   ├── Spring: AuthenticationFilter ───────────────── [15ms]
│   ├── ProjectController.getProject ───────────────── [220ms]
│   │   ├── SELECT FROM projects ───────────────────── [3ms]
│   │   ├── SELECT FROM permissions ────────────────── [2ms]
│   │   └── Redis: GET project:123:settings ────────── [1ms]
│   └── Spring: ResponseSerializer ─────────────────── [5ms]
```

### How Spans and Traces are Named

When you look at a trace, you'll see span names that might seem cryptic at first. Here's how to decode them:

**Naming conventions by span type:**

| Span Type | Naming Pattern | Example |
|-----------|----------------|---------|
| HTTP Server (inbound) | `HTTP_METHOD route_pattern` | `GET /v2/projects/{projectId}` |
| HTTP Client (outbound) | `HTTP_METHOD` | `GET` |
| Database (JDBC) | `operation table_name` or just `operation` | `SELECT`, `SELECT tolgee.projects` |
| Redis | `command` | `GET`, `SET`, `MGET` |
| Spring Controller | `ClassName.methodName` | `ProjectController.getProject` |
| Repository | `ClassName.methodName` | `ProjectRepository.findById` |

**Understanding the hierarchy:**
- The **root span** is usually the HTTP server span (the incoming request)
- Child spans represent operations that happened during the parent span
- The trace name shown in the list is the **root span's name**

**Traces vs Spans in search results:**
- A **trace** is always the complete tree from root to leaves
- When you search by span attribute (e.g., `{span.db.statement=~".*SELECT.*"}`), you get **traces that contain matching spans**, not the individual spans themselves
- To analyze individual spans (e.g., for statistics), see "Switching to Span View" in Part C

### Traces are Always "Top-Level"

A common source of confusion: **there are no "nested traces"**.

- A **trace** is the complete tree representing one request's journey
- **Spans** are the nodes within that tree
- Every trace has exactly one **root span** (the entry point, usually an HTTP request)
- Child spans have a `parentSpanId` linking them to their parent

When you query Tempo, you always get traces (complete trees), not individual spans floating around. The spans you see are always shown in context of their parent trace.

### When Tracing Helps vs. When It Doesn't

**Tracing helps when:**
- Debugging slow requests ("where is the time going?")
- Understanding request flow ("what calls what?")
- Finding failed operations ("what exactly failed?")
- Identifying N+1 queries ("why are there 100 database calls?")
- Correlating issues across services

**Tracing doesn't help when:**
- The problem is not request-related (memory leak, CPU spike from background job)
- You need aggregate metrics ("what's the average response time?") - use metrics instead
- The issue is in code that isn't instrumented

### How Traces Get to Grafana

```
Your App → OpenTelemetry Agent → OTel Collector → Tempo → Grafana
              (creates traces)    (samples/filters)  (stores)  (visualizes)
```

**Important:** Not all traces are stored. The OTel Collector uses **tail sampling**:
- All errors are kept
- All requests >2s are kept
- 10% of other requests are sampled randomly

This means: if you're debugging a specific request, you may not find its trace unless it was slow or failed.

---

## Part B: Navigating Grafana

### Opening Grafana

1. Start the tracing stack:
   ```bash
   cd docker && docker compose -f docker-compose.tracing.yaml up -d
   ```

2. Open http://localhost:3000 in your browser

3. No login required (anonymous access enabled for local development)

![Grafana home screen](images/01-grafana-home.png)

### Selecting Tempo (Trace Storage)

1. Click the **hamburger menu** (☰) in the top left
2. Click **Explore**
3. In the data source dropdown at the top, select **Tempo**

![Explore with Tempo selected](images/02-explore-tempo.png)

### Understanding the Explore View

The Explore view has three main areas:

1. **Query builder** (top): Where you search for traces
2. **Trace list** (middle): Results matching your query
3. **Trace detail** (bottom): Full visualization of a selected trace

![Explore layout](images/03-explore-layout.png)

### Reading the Trace List

The trace list shows matching traces with columns:

| Column | Meaning |
|--------|---------|
| Trace ID | Unique identifier (click to select) |
| Trace Name | The root span's operation name |
| Duration | Total time for the trace |
| Start time | When the request started |
| Span count | Number of operations in the trace |

![Trace list](images/04-trace-list.png)

### Viewing a Trace

Click a trace ID to open the detail view.

#### Waterfall View (default)
Each bar represents a span:
- Horizontal position = when it started
- Bar length = duration
- Color = different services or error status (red for errors)
- Indentation = parent-child relationship

**How to read it:**
- Find the longest bar = the slowest operation
- Look for gaps = waiting time (network latency, blocking)
- Look for many small bars = potentially N+1 queries
- Look for red bars = errors

![Trace waterfall](images/05-trace-waterfall.png)

#### Span Details
Click any span to see:
- Attributes (metadata like HTTP status, SQL query, etc.)
- Events (logs within the span)
- Resource (service info)

![Span details](images/06-span-details.png)

### The Node Graph View

For traces involving multiple services, the Node Graph shows:
- Services as circles
- Requests as arrows between them
- Call counts and latencies on arrows

![Node graph](images/07-node-graph.png)

---

## Part C: Finding What You Need

### Basic TraceQL Queries

TraceQL is the query language for Tempo. Here are the most useful queries:

#### Find all traces from a service
```
{resource.service.name="tolgee-platform"}
```

#### Find traces for a specific endpoint
```
{span.http.route="/v2/projects/{projectId}"}
```

#### Find traces with errors
```
{status=error}
```

#### Find slow traces (>1 second)
```
{duration>1s}
```

#### Combine conditions
```
{resource.service.name="tolgee-platform" && status=error}
{resource.service.name="tolgee-platform" && duration>500ms}
```

![TraceQL query](images/08-traceql-query.png)

### Filtering by Service

When you have multiple services, filter by service name:
```
{resource.service.name="tolgee-platform"}
```

Service names come from the `OTEL_SERVICE_NAME` environment variable.

### Filtering by Operation Name

Find traces for specific API endpoints:
```
{name="GET /v2/projects"}
```

For Spring MVC, the route is templated:
```
{span.http.route="/v2/projects/{projectId}/keys"}
```

### Finding Slow Requests

**All slow traces:**
```
{duration>2s}
```

**Slow database queries:**
```
{span.db.system="postgresql" && duration>100ms}
```

**Slow external HTTP calls:**
```
{span.http.method="GET" && kind=client && duration>1s}
```

### Finding Errors

**Any error:**
```
{status=error}
```

**HTTP 5xx errors:**
```
{span.http.status_code>=500}
```

**Database errors:**
```
{span.db.system="postgresql" && status=error}
```

### Searching by Trace ID

If you have a trace ID from logs or error reports:

1. In Explore, switch to "Search" tab
2. Paste the trace ID
3. Click "Run query"

![Trace ID search](images/09-trace-id-search.png)

### Switching to Span View (Individual Spans vs Traces)

By default, TraceQL queries return **traces** - complete trees containing all spans. This means:
- Query `{span.db.statement=~".*SELECT.*"}` returns traces that **contain** a matching database span
- You see the entire trace, not just the matching span
- Statistics (duration, count) are for the whole trace, not the individual span

**To view individual spans instead:**

1. In the Explore view with Tempo selected
2. Look for the **"Table" dropdown** near the results
3. Switch from "Traces" to "Spans"

Now your results show individual spans with their own durations, making it possible to:
- See statistics for specific operations (e.g., just database queries)
- Filter to only the spans you care about
- Analyze span-level patterns

**Example: Finding slow database queries**

In Trace view:
```
{span.db.system="postgresql" && duration>100ms}
```
→ Returns traces where *any* span took >100ms (might be the HTTP request, not the DB query)

In Span view (after switching):
```
{span.db.system="postgresql" && duration>100ms}
```
→ Returns only the database spans that took >100ms

![Span view toggle](images/16-span-view-toggle.png)

### Common Span Attributes Reference

| Attribute | Description | Example |
|-----------|-------------|---------|
| `service.name` | Service identifier | `tolgee-platform` |
| `http.method` | HTTP method | `GET`, `POST` |
| `http.route` | URL pattern | `/v2/projects/{id}` |
| `http.status_code` | Response status | `200`, `500` |
| `db.system` | Database type | `postgresql`, `redis` |
| `db.statement` | SQL query | `SELECT * FROM...` |
| `db.operation` | Operation type | `SELECT`, `INSERT` |
| `exception.type` | Exception class | `NullPointerException` |
| `exception.message` | Error message | `Project not found` |

---

## Part D: Real-World Use Cases

### Use Case 1: Debugging a Slow API Endpoint

**Scenario:** Users report the `/v2/projects/{id}/keys` endpoint is slow.

**Steps:**

1. Query for slow traces on that endpoint:
   ```
   {span.http.route="/v2/projects/{projectId}/keys" && duration>1s}
   ```

2. Select a slow trace

3. Look at the waterfall:
   - Find the longest bars
   - Check if it's database, Redis, or application code

4. Common findings:
   - **Long database query:** Check the `db.statement` attribute
   - **Many database queries:** Count the SELECT spans (might be N+1)
   - **Long gaps:** Network latency or blocking code

![Slow endpoint](images/10-slow-endpoint.png)

### Use Case 2: Finding Why a Request Failed

**Scenario:** A user got a 500 error and you have the timestamp.

**Steps:**

1. Query for errors around that time:
   ```
   {status=error}
   ```
   (Set time range to around the incident)

2. Or, if you have the trace ID from logs:
   ```
   {trace:THE_TRACE_ID}
   ```

3. Find the red span (error) in the waterfall

4. Click the span, check:
   - `exception.type`
   - `exception.message`
   - `exception.stacktrace` (if available)

![Error trace](images/11-error-trace.png)

### Use Case 3: Understanding Request Flow

**Scenario:** You're new to the codebase and want to understand what happens when a user creates a key.

**Steps:**

1. Make the request yourself:
   ```bash
   curl -X POST http://localhost:8080/v2/projects/123/keys \
     -H "Content-Type: application/json" \
     -d '{"name": "test-key"}'
   ```

2. Find the trace:
   ```
   {span.http.route="/v2/projects/{projectId}/keys" && span.http.method="POST"}
   ```

3. Examine the waterfall to see:
   - What controllers are called
   - What database tables are accessed
   - What services are invoked

![Request flow](images/12-request-flow.png)

### Use Case 4: Correlating Traces with Logs

**Scenario:** You see an error in the logs and want to find the full trace context.

**Steps:**

1. In Grafana, go to Explore → Loki
2. Search for the error:
   ```
   {container_name=~".*tolgee.*"} |= "error"
   ```
3. Find the log line with trace ID
4. Click "View Trace" link (automatically created by log-trace correlation)

Or vice versa:

1. Find a trace in Tempo
2. Click "Logs for this span" button
3. See all logs emitted during that span

![Trace-log correlation](images/13-trace-log-correlation.png)

### Use Case 5: Identifying N+1 Database Queries

**Scenario:** An endpoint is slow and you suspect N+1 queries.

**Steps:**

1. Find a trace for the endpoint
2. Count the database spans
3. If you see many identical queries (e.g., 50 SELECT FROM keys):
   - Pattern confirmed: N+1 query
   - Each child entity is being loaded separately

**What to look for:**
```
├── GET /v2/projects/123/keys ───────────────── [2500ms]
│   ├── SELECT FROM projects ─────────────────── [5ms]
│   ├── SELECT FROM keys WHERE project_id=123 ── [20ms]
│   ├── SELECT FROM key_metas WHERE id=1 ─────── [3ms]  ┐
│   ├── SELECT FROM key_metas WHERE id=2 ─────── [3ms]  │
│   ├── SELECT FROM key_metas WHERE id=3 ─────── [3ms]  │ N+1!
│   ├── ... (47 more) ────────────────────────── [3ms]  │
│   └── SELECT FROM key_metas WHERE id=50 ────── [3ms]  ┘
```

**Fix:** Use batch loading or eager fetching.

![N+1 pattern](images/14-n-plus-one.png)

---

## Part E: Span Metrics and Time-Series Analysis

### The Problem: Traces vs Metrics

Tracing shows you **individual requests** in detail. But what if you want to answer:
- "What's the p95 latency for this database query over the last hour?"
- "How has the average response time for this endpoint changed over time?"
- "What's the error rate trend for this service?"

These are **metrics** questions, not tracing questions. Traces give you individual data points; metrics give you aggregates over time.

### Current Limitations

**Tempo (our trace storage) is not a metrics database.** It stores individual traces, not time-series aggregates. This means:

- You cannot directly plot "p95 query time over last 24 hours" in Tempo
- TraceQL gives you lists of traces/spans, not statistical aggregates
- The Explore view shows individual results, not charts over time

### Option 1: Export Span Metrics to Prometheus (Requires Setup)

Tempo can generate RED metrics (Rate, Errors, Duration) from spans and export them to Prometheus. This is configured in `tempo-config.yaml`:

```yaml
overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics]
```

**What this enables:**
- Automatic generation of `traces_spanmetrics_latency_bucket` histogram
- Rate metrics per operation
- Error rate metrics

**What's needed to use this:**
1. Add Prometheus to the tracing stack
2. Configure Tempo to export metrics to Prometheus
3. Create Grafana dashboards using Prometheus as datasource

**This is out of scope for Phase 2** but documented here for future reference. If you need time-series analysis of span metrics, this is the path forward.

### Option 2: Manual Analysis with Span View

For ad-hoc analysis without additional infrastructure:

1. Switch to Span view (see "Switching to Span View" above)
2. Query for the spans you're interested in:
   ```
   {span.db.statement=~".*SELECT.*FROM.*projects.*" && span.db.system="postgresql"}
   ```
3. Export results (if available) or manually note durations
4. Use external tools (spreadsheet, script) for statistical analysis

**Limitations:**
- Manual and time-consuming
- Only works for recent data (within trace retention window)
- Not suitable for ongoing monitoring

### Option 3: Loki Log Metrics (If Logs Contain Durations)

If your application logs include timing information, you can use Loki's metric queries:

```
sum(rate({container_name="tolgee"} |= "query completed" | regexp `duration=(?P<duration>\d+)ms` | unwrap duration [5m])) by (query)
```

**Limitations:**
- Only works if you log the data you need
- Less accurate than span-based metrics
- Requires log parsing

### Recommendation

For Phase 2 (documentation focus), we document:
1. What's possible with current setup (trace-level analysis)
2. What's NOT possible without additional infrastructure (time-series metrics)
3. The path to enable metrics if needed in the future

**Future enhancement (Phase 3?):** Add Prometheus to the tracing stack and create dashboards for span-based metrics.

![Metrics limitation note](images/17-metrics-limitation.png)

---

## Quick Reference

### Start Tracing Stack
```bash
cd docker && docker compose -f docker-compose.tracing.yaml up -d
```

### Run App with Tracing
```bash
OTEL_JAVAAGENT_ENABLED=true \
OTEL_SERVICE_NAME=tolgee-platform \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 \
OTEL_LOGS_EXPORTER=none \
OTEL_METRICS_EXPORTER=none \
./gradlew server-app:bootRun --args='--spring.profiles.active=dev'
```

### Common TraceQL Queries

| Goal | Query |
|------|-------|
| All traces from service | `{resource.service.name="tolgee-platform"}` |
| Errors | `{status=error}` |
| Slow requests | `{duration>2s}` |
| Specific endpoint | `{span.http.route="/v2/projects/{projectId}"}` |
| Database queries | `{span.db.system="postgresql"}` |
| By trace ID | Search tab or paste ID directly |

### Where to Get Help

- [Grafana Tempo Documentation](https://grafana.com/docs/tempo/latest/)
- [TraceQL Documentation](https://grafana.com/docs/tempo/latest/traceql/)
- [OpenTelemetry Concepts](https://opentelemetry.io/docs/concepts/)
- [Tolgee Tracing Configuration](./otel-collector-config.yaml)
```

### Screenshots

Create `docker/tracing/images/` directory and capture the following screenshots:

| Filename | Description |
|----------|-------------|
| `01-grafana-home.png` | Grafana home screen with menu visible |
| `02-explore-tempo.png` | Explore view with Tempo datasource selected |
| `03-explore-layout.png` | Annotated layout showing query, list, detail areas |
| `04-trace-list.png` | Trace list with columns labeled |
| `05-trace-waterfall.png` | Expanded trace showing parent-child spans with naming conventions visible |
| `06-span-details.png` | Span clicked, showing attributes panel |
| `07-node-graph.png` | Node graph visualization |
| `08-traceql-query.png` | Query builder with a sample TraceQL query |
| `09-trace-id-search.png` | Search tab with trace ID entered |
| `10-slow-endpoint.png` | Slow span highlighted in waterfall |
| `11-error-trace.png` | Error trace with red span and exception details |
| `12-request-flow.png` | Clean waterfall showing request lifecycle |
| `13-trace-log-correlation.png` | Logs panel linked from trace (requires Step 0.5 complete) |
| `14-n-plus-one.png` | Many similar database spans showing N+1 pattern |
| `15-log-trace-link.png` | Loki log with "View Trace" link highlighted |
| `16-span-view-toggle.png` | Dropdown showing Traces vs Spans view toggle |
| `17-metrics-limitation.png` | Annotated screenshot explaining metrics vs traces difference |

### Testing
- Tutorial renders correctly in GitHub/VS Code markdown preview
- Screenshots display properly (relative paths work)
- All TraceQL examples work against local Tempo

### Checklist
- [ ] Tutorial file created at `docker/tracing/GRAFANA_TUTORIAL.md`
- [ ] Part A: Tracing Fundamentals complete (including span naming, top-level explanation)
- [ ] Part B: Navigating Grafana UI complete
- [ ] Part C: Finding What You Need complete (including span view toggle)
- [ ] Part D: Real-World Use Cases complete
- [ ] Part E: Span Metrics and Time-Series Analysis complete
- [ ] All 17 screenshots taken and saved to `docker/tracing/images/`
- [ ] **Log entry appended to PHASE_2_LOG.md**

---

## Step 2: Add Entry Point Documentation

### Goal
Add documentation comments to docker-compose.tracing.yaml that explain the system and link to the tutorial.

### Prerequisites
- Tutorial created (Step 1)

### Instructions

Add the following header comment to the top of `docker/docker-compose.tracing.yaml` (before the `name:` line):

```yaml
# =============================================================================
# Local Tracing Stack for Tolgee Development
# =============================================================================
#
# This stack provides distributed tracing infrastructure for debugging:
# - Slow requests (where is the time going?)
# - Failed requests (what exactly failed?)
# - Request flow (what calls what?)
#
# QUICK START:
#   docker compose -f docker-compose.tracing.yaml up -d
#   open http://localhost:3000  # Grafana
#
# FULL TUTORIAL: See tracing/GRAFANA_TUTORIAL.md
#
# COMPONENTS:
#   - Tempo: Trace storage (port 3200)
#   - OTel Collector: Trace pipeline with tail sampling (ports 4317/4318)
#   - Loki: Log storage with trace correlation (port 3100)
#   - Grafana: Visualization UI (port 3000)
#   - PostgreSQL/Redis: Tolgee dependencies
#
# CONFIGURATION FILES:
#   - tracing/tempo-config.yaml - Tempo storage and query settings
#   - tracing/otel-collector-config.yaml - Sampling policies (errors, slow, 10%)
#   - tracing/loki-config.yaml - Log ingestion settings
#   - tracing/grafana/datasources.yaml - Pre-configured data sources
#
# =============================================================================
```

### Testing
- Header comment visible when opening file
- Links reference correct file paths

### Checklist
- [ ] Header comment added to docker-compose.tracing.yaml
- [ ] Links to GRAFANA_TUTORIAL.md
- [ ] Links to configuration files with brief purpose
- [ ] **Log entry appended to PHASE_2_LOG.md**

---

## Step 3: Add Inline Documentation to Configuration Files

### Goal
Add explanatory comments to each configuration file explaining what the settings do.

### Prerequisites
- None

### Instructions

Add header comments and inline explanations to:

**1. `docker/tracing/tempo-config.yaml`**
- Explain storage settings
- Explain retention period
- Explain query frontend settings

**2. `docker/tracing/otel-collector-config.yaml`**
- Explain receivers (OTLP gRPC/HTTP)
- Explain tail sampling policies (why errors, why slow, why 10%)
- Explain batch processor settings
- Explain exporter configuration

**3. `docker/tracing/loki-config.yaml`**
- Explain log storage settings
- Explain retention settings

**4. `docker/tracing/promtail-config.yaml`**
- Explain Docker log collection
- Explain trace ID extraction regex

**5. `docker/tracing/grafana/datasources.yaml`**
- Explain each datasource
- Explain trace-to-log correlation settings

### Testing
- Each config file opens and parses correctly (no YAML errors)
- Comments are helpful for understanding the configuration

### Checklist
- [ ] tempo-config.yaml documented
- [ ] otel-collector-config.yaml documented
- [ ] loki-config.yaml documented
- [ ] promtail-config.yaml documented
- [ ] datasources.yaml documented
- [ ] **Log entry appended to PHASE_2_LOG.md**

---

## Step 4: Update DEVELOPMENT.md with Tracing Section

### Goal
Add a section to DEVELOPMENT.md that introduces tracing and links to the tutorial.

### Prerequisites
- Tutorial exists (Step 1)

### Instructions

Add a "Tracing" section to `DEVELOPMENT.md` (find appropriate location after existing sections):

```markdown
## Tracing

For debugging slow requests, failed operations, and understanding request flow, Tolgee includes a local tracing stack with Grafana, Tempo, and OpenTelemetry.

### Quick Start

1. Start the tracing stack:
   ```bash
   cd docker && docker compose -f docker-compose.tracing.yaml up -d
   ```

2. Start the backend with tracing enabled:
   ```bash
   OTEL_JAVAAGENT_ENABLED=true \
   OTEL_SERVICE_NAME=tolgee-platform \
   OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 \
   OTEL_LOGS_EXPORTER=none \
   OTEL_METRICS_EXPORTER=none \
   ./gradlew server-app:bootRun --args='--spring.profiles.active=dev'
   ```

3. Open Grafana at http://localhost:3000 and go to Explore → Tempo

For a complete guide including TraceQL queries and debugging use cases, see the [Grafana Tracing Tutorial](docker/tracing/GRAFANA_TUTORIAL.md).
```

### Testing
- Section appears in DEVELOPMENT.md
- Links work from repository root

### Checklist
- [ ] Tracing section added to DEVELOPMENT.md
- [ ] Quick-start commands included
- [ ] Link to tutorial is correct
- [ ] **Log entry appended to PHASE_2_LOG.md**

---

## Step 5: Verify Documentation Hierarchy

### Goal
Walk through the documentation as a newcomer to verify it's navigable.

### Prerequisites
- All previous steps complete

### Instructions

1. Start from DEVELOPMENT.md
2. Follow links to tracing documentation
3. Verify tutorial is understandable
4. Check all cross-references work
5. Verify screenshots render in markdown preview

### Testing
- Full path from DEVELOPMENT.md → Tutorial works
- docker-compose.tracing.yaml → Tutorial link works
- All internal links working
- Screenshots display correctly

### Checklist
- [ ] Path verified: DEVELOPMENT.md → Tutorial
- [ ] Path verified: docker-compose.tracing.yaml → Tutorial
- [ ] All internal links working
- [ ] Screenshots render correctly
- [ ] **Log entry appended to PHASE_2_LOG.md**

---

## Rollback

```bash
# Remove created documentation files
rm -f docs/implementation-log/2026-01-16-backend-tracing/DOCS_PLAN.md
rm -f docker/tracing/GRAFANA_TUTORIAL.md
rm -f docker/tracing/README.md
rm -rf docker/tracing/images/

# Revert modified files
git checkout -- docker/docker-compose.tracing.yaml
git checkout -- docker/tracing/tempo-config.yaml
git checkout -- docker/tracing/otel-collector-config.yaml
git checkout -- docker/tracing/loki-config.yaml
git checkout -- docker/tracing/promtail-config.yaml
git checkout -- docker/tracing/grafana/datasources.yaml
git checkout -- DEVELOPMENT.md
```

---

## References

- [Grafana Tempo Documentation](https://grafana.com/docs/tempo/latest/)
- [TraceQL Documentation](https://grafana.com/docs/tempo/latest/traceql/)
- [OpenTelemetry Concepts](https://opentelemetry.io/docs/concepts/)
- [OpenTelemetry Java Agent](https://opentelemetry.io/docs/zero-code/java/agent/)

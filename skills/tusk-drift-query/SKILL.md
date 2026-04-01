---
name: tusk-drift-query
description: Search, analyze, and debug recorded API traffic using the Tusk Drift CLI query commands. Use when users want to explore their API endpoints, investigate errors or latency, trace requests, or understand traffic patterns.
---

Search and analyze API traffic span recordings from Tusk Drift using CLI commands.

These commands help you query, analyze, and debug API traffic including HTTP requests/responses, database queries, gRPC calls, and more — latency metrics, error rates, and distributed traces across services.

All commands output JSON to stdout. All accept `--service-id` to override the default from `.tusk/config.yaml`.

## Service discovery

Before running any query commands, run `tusk drift query services` to find available services.

- **0 services**: The user hasn't set up Tusk Drift cloud yet. Tell them to run `tusk drift setup` in the service of their choosing.
- **1 service**: Use that service's `id` as `--service-id` on all subsequent commands. No need to ask the user.
- **Multiple services**: STOP and ask the user which service they want to query before running any other commands. Present the list with names and repo info. Use their choice as `--service-id` on all subsequent commands.

If `tusk drift query services` fails with an auth error, tell the user to run `tusk auth login` or set `TUSK_API_KEY`.

## Workflow tips

- Start with `tusk drift query distinct` to discover available endpoints
- Use `tusk drift query spans` to find specific API calls
- Use `tusk drift query trace` to debug a request's full call chain

### Root cause analysis workflow

If the user is investigating performance issues or errors:

1. Use `spans` or `aggregate` to identify the problematic endpoint/span
2. Use `trace` to see the full call chain and identify which child span is the bottleneck
3. Look at the span's metadata (inputValue/outputValue) to understand the request context
4. Navigate to the relevant source code using the span name (usually maps to route handlers or functions)
5. Analyze the code path to understand the root cause

---

## Commands

### `tusk drift query distinct`

List unique values for a field, ordered by frequency.

Use this to:

- Discover available endpoints: `--field name`
- See all instrumentation packages in use: `--field packageName`
- Find unique environments: `--field environment`
- Explore JSONB values like status codes: `--field "outputValue.statusCode"`

This helps you understand what values exist before building specific queries.

**Flags:**

- `--field` (required) — Field to get distinct values for. Can be a column name or JSONB path (e.g., `name`, `packageName`, `outputValue.statusCode`)
- `--limit` (default 50) — Max distinct values to return (1-100)
- `--where` — Optional filter conditions as JSON (SpanWhereClause)
- `--jsonb-filters` — Optional JSONB filters as JSON array

---

### `tusk drift query spans`

Search and filter API traffic span recordings.

Use this to:

- Find specific API calls by endpoint name, HTTP method, or status code
- Search for errors or slow requests
- Get recent traffic for a specific endpoint
- Debug specific API calls

Examples:

- Find failed requests: `--where '{"name":{"contains":"/api/users"}}' --jsonb-filters '[{"column":"outputValue","jsonPath":"$.statusCode","gte":400,"castAs":"int"}]'`
- Find slow requests: `--min-duration 1000`
- Recent traffic for endpoint: `--name "/api/orders" --limit 10 --order-by timestamp:DESC`

**Convenience flags:**

- `--name` — Filter by span/endpoint name (exact match)
- `--package-name` — Filter by instrumentation package (http, pg, fetch, grpc, etc.)
- `--trace-id` — Filter by trace ID
- `--environment` — Filter by environment
- `--min-duration` — Minimum duration in milliseconds
- `--root-spans-only` — Only return root spans
- `--limit` (default 20) — Max results (1-100)
- `--offset` (default 0) — Pagination offset
- `--include-payloads` (default false) — Include full inputValue/outputValue (can be verbose)
- `--max-payload-length` (default 500) — Truncate payload strings to this length
- `--order-by` — Sort results, format `field:direction` (e.g., `timestamp:DESC`, `duration:ASC`). Fields: timestamp, duration, name

**Advanced flags (JSON strings, override convenience flags):**

- `--where` — Full SpanWhereClause as JSON. Supports field filters (name, packageName, instrumentationName, environment, traceId, spanId, duration, isRootSpan) with operators (eq, neq, in, contains, startsWith, endsWith, gt, gte, lt, lte). Supports AND/OR arrays for combining conditions.
- `--jsonb-filters` — Array of JSONB filters as JSON. Each filter has: `column` (inputValue, outputValue, metadata, status), `jsonPath` (starting with $, e.g., `$.statusCode`), comparison operators (eq, neq, gt, gte, lt, lte, contains, startsWith, endsWith, isNull, in), and optional `castAs` (text, int, float, boolean).

---

### `tusk drift query trace <trace-id>`

Get all spans in a distributed trace as a hierarchical tree.

Use this to:

- Debug a specific request end-to-end
- See the full call chain from HTTP request to database queries
- Understand timing and dependencies between spans
- Identify bottlenecks in a request

First use `spans` to find spans, then use the traceId from the results to get the full trace.

**Flags:**

- `--include-payloads` (default false) — Include inputValue/outputValue
- `--max-payload-length` (default 500) — Truncate payload strings to this length

---

### `tusk drift query aggregate`

Calculate aggregated metrics and statistics across spans.

Use this to:

- Get latency percentiles for endpoints (p50, p95, p99)
- Calculate error rates by endpoint
- Get request counts over time
- Compare performance across environments

Examples:

- Endpoint latency: `--group-by name --metrics count,avgDuration,p95Duration`
- Error rates: `--group-by name --metrics count,errorCount,errorRate`
- Hourly trends: `--time-bucket hour --metrics count,errorRate`

**Flags:**

- `--metrics` (required) — Metrics to calculate, comma-separated. Values: count, errorCount, errorRate, avgDuration, minDuration, maxDuration, p50Duration, p95Duration, p99Duration
- `--group-by` — Fields to group by, comma-separated. Values: name, packageName, instrumentationName, environment, statusCode
- `--time-bucket` — Time bucket for time-series data: hour, day, week
- `--order-by` — Order by metric, format `metric:direction` (e.g., `count:DESC`)
- `--limit` (default 20) — Max results (1-100)
- `--where` — Filter conditions as JSON (SpanWhereClause)

---

### `tusk drift query schema`

Get schema and structure information for span recordings.

Use this to:

- Understand what fields are available for a specific instrumentation type
- See example payloads for HTTP requests, database queries, etc.
- Learn what to filter on before querying spans

Common package names:

- http: Incoming HTTP requests (has statusCode, method, url, headers)
- fetch: Outgoing HTTP calls
- pg: PostgreSQL queries (has db.statement, db.name)
- grpc: gRPC calls
- express: Express.js middleware spans

**Flags:**

- `--name` — Span name to get schema for (e.g., `/api/users`)
- `--package-name` — Package name (e.g., http, pg, fetch)
- `--instrumentation-name` — Specific instrumentation name
- `--show-example` (default true) — Include an example span with real data
- `--max-payload-length` (default 500) — Truncate example payload strings

---

### `tusk drift query spans-by-ids`

Fetch specific span recordings by their IDs.

Use this when you have span IDs from a previous query and need the full details including payloads. Useful for:

- Getting full details for spans found via `spans`
- Examining specific requests in detail
- Comparing multiple specific spans

**Flags:**

- `--ids` (required) — Span recording IDs, comma-separated (max 20)
- `--include-payloads` (default true) — Include full inputValue/outputValue
- `--max-payload-length` (default 500) — Truncate payload strings to this length

---

## Error handling

- **Auth errors** (401/403): Tell the user to run `tusk auth login` or set `TUSK_API_KEY`.
- **No data returned**: The service may not have recorded traces yet. Direct them to https://docs.usetusk.ai/api-tests/record-replay-tests.

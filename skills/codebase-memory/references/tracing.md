# Call Chain Tracing

One `trace_call_path` call replaces dozens of grep searches across files.

## Step 1: Discover the exact function name

`trace_call_path` requires an **exact** name match. Discover it first with regex:

```
search_graph(name_pattern=".*Order.*", label="Function")
```

Useful regex patterns:
- `(?i)order` — case-insensitive
- `^(Get|Set|Delete)Order` — CRUD variants
- `.*Order.*Handler$` — handlers only
- `qn_pattern=".*services\\.order\\..*"` — scope to directory

## Step 2: Trace callers (who calls this?)

```
trace_call_path(function_name="ProcessOrder", direction="inbound", depth=3)
```

## Step 3: Trace callees (what does this call?)

```
trace_call_path(function_name="ProcessOrder", direction="outbound", depth=3)
```

## Step 4: Full context (both)

```
trace_call_path(function_name="ProcessOrder", direction="both", depth=3)
```

**Always use `direction="both"` for complete context.** Cross-service HTTP_CALLS edges appear as inbound edges — `direction="outbound"` alone misses them.

## Read suspicious code

```
get_code_snippet(qualified_name="project.path.module.FunctionName")
```

## Cross-Service HTTP Calls

See all HTTP links with URLs and confidence:
```
query_graph(query="MATCH (a)-[r:HTTP_CALLS]->(b) RETURN a.name, b.name, r.url_path, r.confidence ORDER BY r.confidence DESC LIMIT 20")
```

Filter by URL:
```
query_graph(query="MATCH (a)-[r:HTTP_CALLS]->(b) WHERE r.url_path CONTAINS '/orders' RETURN a.name, b.name, r.url_path")
```

## Async Dispatch (Cloud Tasks, Pub/Sub, etc.)

```
search_graph(name_pattern=".*CreateTask.*|.*send_to_pubsub.*")
trace_call_path(function_name="CreateMultidataTask", direction="both")
```

## Interface Implementations

```
query_graph(query="MATCH (s)-[r:OVERRIDE]->(i) WHERE i.name = 'Read' RETURN s.name, i.name LIMIT 20")
```

## Read References (callbacks, variable assignments)

```
query_graph(query="MATCH (a)-[r:USAGE]->(b) WHERE b.name = 'ProcessOrder' RETURN a.name, a.file_path LIMIT 20")
```

## Risk-Classified Impact Analysis

```
trace_call_path(function_name="ProcessOrder", direction="inbound", depth=3, risk_labels=true)
```

Returns nodes with `risk` (CRITICAL/HIGH/MEDIUM/LOW) based on hop depth. Hop 1=CRITICAL, 2=HIGH, 3=MEDIUM, 4+=LOW.

## Detect Changes (Git Diff Impact)

```
detect_changes()
detect_changes(scope="staged")
detect_changes(scope="branch", base_branch="main")
```

Returns changed files, changed symbols, and impacted callers with risk classification. Scopes: `unstaged`, `staged`, `all` (default), `branch`.

## Tips

- Start with `depth=1` for quick answers, increase only if needed (max 5).
- Edge types in traces: `CALLS` (direct), `HTTP_CALLS` (cross-service), `ASYNC_CALLS` (async), `USAGE` (read reference), `OVERRIDE` (interface implementation).
- Results are capped at 200 nodes per trace.
- `detect_changes` requires git in PATH.

# Codebase Memory MCP — Tool Reference

## Tools (14 total)

| Tool | Purpose |
|------|---------|
| `index_repository` | Parse and ingest repo into graph (only once — auto-sync keeps it fresh) |
| `index_status` | Check indexing status (ready/indexing/not found) |
| `list_projects` | List all indexed projects with timestamps and counts |
| `delete_project` | Remove a project from the graph |
| `search_graph` | Structured search with filters (name, label, degree, file pattern) |
| `search_code` | Grep-like text search within indexed project files |
| `trace_call_path` | BFS call chain traversal (exact name match required). Supports `risk_labels=true`. |
| `detect_changes` | Map git diff to affected symbols + blast radius with risk scoring |
| `query_graph` | Cypher-like graph queries (200-row cap) |
| `get_graph_schema` | Node/edge counts, relationship patterns |
| `get_code_snippet` | Read source code by qualified name |
| `read_file` | Read any file from indexed project |
| `list_directory` | List files/directories with glob filter |
| `ingest_traces` | Ingest OpenTelemetry traces to validate HTTP_CALLS edges |

## Edge Types

| Type | Meaning |
|------|---------|
| `CALLS` | Direct function call within same service |
| `HTTP_CALLS` | Synchronous cross-service HTTP request |
| `ASYNC_CALLS` | Async dispatch (Cloud Tasks, Pub/Sub, SQS, Kafka) |
| `IMPORTS` | Module/package import |
| `DEFINES` / `DEFINES_METHOD` | Module/class defines a function/method |
| `HANDLES` | Route node handled by a function |
| `IMPLEMENTS` | Type implements an interface |
| `OVERRIDE` | Struct method overrides an interface method |
| `USAGE` | Read reference (callback, variable assignment) |
| `FILE_CHANGES_WITH` | Git history change coupling |
| `CONTAINS_FILE` / `CONTAINS_FOLDER` / `CONTAINS_PACKAGE` | Structural containment |

## Node Labels

`Project`, `Package`, `Folder`, `File`, `Module`, `Class`, `Function`, `Method`, `Interface`, `Enum`, `Type`, `Route`

## Qualified Name Format

`<project>.<path_parts>.<name>` — file path with `/` replaced by `.`, extension removed.

Examples:
- `myproject.cmd.server.main.HandleRequest` (Go)
- `myproject.services.orders.ProcessOrder` (Python)
- `myproject.src.components.App.App` (TypeScript)

Use `search_graph` to discover qualified names, then pass them to `get_code_snippet`.

## Cypher Subset (for query_graph)

**Supported:**
- `MATCH` with node labels and relationship types
- Variable-length paths: `-[:CALLS*1..3]->`
- `WHERE` with `=`, `<>`, `>`, `<`, `>=`, `<=`, `=~` (regex), `CONTAINS`, `STARTS WITH`
- `WHERE` with `AND`, `OR`, `NOT`
- `RETURN` with property access, `COUNT(x)`, `DISTINCT`
- `ORDER BY` with `ASC`/`DESC`
- `LIMIT`
- Edge property access: `r.confidence`, `r.url_path`, `r.coupling_score`

**Not supported:** `WITH`, `COLLECT`, `SUM`, `CREATE/DELETE/SET`, `OPTIONAL MATCH`, `UNION`

## Common Cypher Patterns

```
# Cross-service HTTP calls with confidence
MATCH (a)-[r:HTTP_CALLS]->(b) RETURN a.name, b.name, r.url_path, r.confidence LIMIT 20

# Filter by URL path
MATCH (a)-[r:HTTP_CALLS]->(b) WHERE r.url_path CONTAINS '/orders' RETURN a.name, b.name

# Interface implementations
MATCH (s)-[r:OVERRIDE]->(i) RETURN s.name, i.name LIMIT 20

# Change coupling
MATCH (a)-[r:FILE_CHANGES_WITH]->(b) WHERE r.coupling_score >= 0.5 RETURN a.name, b.name, r.coupling_score

# Functions calling a specific function
MATCH (f:Function)-[:CALLS]->(g:Function) WHERE g.name = 'ProcessOrder' RETURN f.name LIMIT 20
```

## Regex Patterns for search_graph

| Pattern | Matches | Use case |
|---------|---------|----------|
| `.*Handler$` | names ending in Handler | Find all handlers |
| `(?i)auth` | case-insensitive "auth" | Find auth-related symbols |
| `get\|fetch\|load` | any of three words | Find data-loading functions |
| `^on[A-Z]` | names starting with on + uppercase | Find event handlers |
| `.*Service.*Impl` | Service...Impl pattern | Find service implementations |
| `^(Get\|Set\|Delete)` | CRUD prefixes | Find CRUD operations |
| `.*\\.controllers\\..*` | qn_pattern for directory scoping | Scope to controllers dir |

## Regex Patterns for search_code

| Pattern | Matches | Use case |
|---------|---------|----------|
| `TODO\|FIXME\|HACK` | multi-pattern scan | Find tech debt markers |
| `(?i)password\|secret\|token` | case-insensitive secrets | Security scan |
| `func\\s+Test` | Go test functions | Find test entry points |
| `api[._/]v[0-9]` | API version references | Find versioned API usage |

## Combining Filters

```
# Find unused auth handlers
search_graph(name_pattern="(?i).*auth.*handler.*", max_degree=0, exclude_entry_points=true)

# Find high fan-out functions in services directory
search_graph(qn_pattern=".*\\.services\\..*", min_degree=10, relationship="CALLS", direction="outbound")

# Find route handlers matching a URL pattern
search_code(pattern="(?i)(POST|PUT).*\\/api\\/v[0-9]\\/orders", regex=true)
```

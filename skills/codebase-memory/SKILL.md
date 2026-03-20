---
name: codebase-memory
description: >
  Use the codebase knowledge graph for structural code queries. Triggers on: "explore the codebase",
  "understand the architecture", "what functions exist", "show me the structure", "who calls this function",
  "what does X call", "trace the call chain", "find callers of", "show dependencies", "impact analysis",
  "dead code", "unused functions", "high fan-out", "refactor candidates", "code quality audit",
  "graph query syntax", "Cypher query examples", "edge types", "how to use search_graph".
---

# Codebase Memory — Knowledge Graph Tools

Graph tools return precise structural results in ~500 tokens vs ~80K for grep-based exploration.

## What do you want to do?

| Goal | Read |
|------|------|
| **Explore codebase structure** — find functions, classes, routes, understand architecture | [references/exploring.md](references/exploring.md) |
| **Trace call chains** — who calls X, what does X call, impact analysis, cross-service calls | [references/tracing.md](references/tracing.md) |
| **Code quality analysis** — dead code, high fan-out, hidden coupling, refactor candidates | [references/quality.md](references/quality.md) |
| **Tool reference** — all 14 tools, edge types, node labels, Cypher syntax, regex patterns | [references/tool-reference.md](references/tool-reference.md) |

## Quick Decision Matrix

| Question | Tool call |
|----------|-----------|
| Who calls X? | `trace_call_path(direction="inbound")` |
| What does X call? | `trace_call_path(direction="outbound")` |
| Full call context | `trace_call_path(direction="both")` |
| Find by name pattern | `search_graph(name_pattern="...")` |
| Dead code | `search_graph(max_degree=0, exclude_entry_points=true)` |
| Cross-service edges | `query_graph` with Cypher |
| Impact of local changes | `detect_changes()` |
| Risk-classified trace | `trace_call_path(risk_labels=true)` |
| Text search | `search_code` or Grep |

## Gotchas

1. **`search_graph(relationship="HTTP_CALLS")` does NOT return edges** — it filters nodes by degree. Use `query_graph` with Cypher to see actual edge properties (url_path, confidence).
2. **`query_graph` has a 200-row cap** before aggregation — COUNT queries silently undercount on large codebases. Use `search_graph` with `min_degree`/`max_degree` for counting.
3. **`trace_call_path` needs exact names** — use `search_graph(name_pattern=".*Partial.*")` first to discover the exact function name.
4. **`direction="outbound"` misses cross-service callers** — always use `direction="both"` for complete context. Cross-service HTTP_CALLS appear as inbound edges.
5. **Results default to 10 per page** — check `has_more` and use `offset` to paginate.
6. **Dead code detection requires entry point exclusion** — without `exclude_entry_points=true`, route handlers and `main()` show as false positives.
7. **`search_graph` with degree filters has no row cap** (unlike `query_graph`). Use it for counting, not `query_graph`.

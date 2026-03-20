# Codebase Exploration

## Get a structural overview

```
get_graph_schema
```

Returns node label counts (functions, classes, routes), edge type counts, and relationship patterns. Use it to understand what's in the graph before querying.

## Find specific code elements

Find functions by name pattern:
```
search_graph(label="Function", name_pattern=".*Handler.*")
```

Find classes:
```
search_graph(label="Class", name_pattern=".*Service.*")
```

Find all REST routes:
```
search_graph(label="Route")
```

Find modules/packages:
```
search_graph(label="Module")
```

Scope to a specific directory:
```
search_graph(label="Function", qn_pattern=".*services\\.order\\..*")
```

## Read source code

After finding a function via search, read its source:
```
get_code_snippet(qualified_name="project.path.to.FunctionName")
```

## File/directory exploration

```
list_directory(path="src/services")
```

## When to Use Grep Instead

- Searching for **string literals** or error messages → `search_code` or Grep
- Finding a file by exact name → Glob
- The graph indexes structural elements, not text content

## Tips

- Use `project` parameter when multiple repos are indexed.
- Route nodes have a `properties.handler` field with the handler function name.
- `exclude_labels` removes noise (e.g., `exclude_labels=["Route"]` when searching by name pattern).

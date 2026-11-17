# Architecture

## Overview

```
Claude Desktop
    ↓
MCP Client
    ↓
Python MCP Server (FastMCP 3.4.4, stdio or HTTP)
    ↓
Business Analytics Tools (6 tools, 3 resources, 2 prompts)
    ↓
SQLite Database (data/enterprise.db)
    ↓
Structured Tool Results (Pydantic models → JSON Schema outputSchema)
    ↓
Claude Desktop
```

The client (Claude Desktop, MCP Inspector, or any other MCP-compatible app) starts the
Python process, speaks MCP over stdio (or HTTP), and the server translates each request
into a parameterized SQLite query against a seeded "enterprise sales" database.

## File-by-file responsibilities

| File | Responsibility |
|---|---|
| `src/analytics_mcp/database.py` | Schema DDL, deterministic seed-data generator (`random.seed(42)` — same data every time), `query_db()` helper, `get_schema_info()` introspection |
| `src/analytics_mcp/schemas.py` | Pydantic models that become each tool's `outputSchema` (`QueryResult`, `RevenueReport`, `CustomerProfile`, `TopProductsReport`, `TrendAnalysis`, `RegionalReport`) |
| `src/analytics_mcp/server.py` | The `FastMCP("Enterprise Analytics")` instance; registers 6 `@mcp.tool`, 3 `@mcp.resource`, 2 `@mcp.prompt` functions, all evaluated at import time |
| `src/analytics_mcp/cli.py` | Typer CLI: `serve` (stdio/http), `init` (seed DB), `inspect` (lists registered tools/resources/prompts) |
| `tests/` | 43 tests across database logic, direct tool calls, and full in-memory MCP client/server integration |
| `docs/ADR.md` | The original author's own design-decision log |

## Tools vs. Resources vs. Prompts

- **Tools** (`@mcp.tool`) are callable functions with typed inputs/outputs. All 6 here are
  annotated `readOnlyHint: True` — they never write to the database.
- **Resources** (`@mcp.resource`) are addressable, readable content at a URI, like a
  virtual filesystem. Two of the three here are **static** (`schema://database`,
  `report://templates`); the third, `schema://tables/{table_name}`, is a **resource
  template** — its URI takes a parameter.
  - **Important gotcha we found:** the project's own CLI `inspect` command only calls
    `mcp.list_resources()`, which returns the 2 static resources. It does **not** call
    `list_resource_templates()`, so the templated resource never shows up in `inspect`
    output — even though it's fully registered and works correctly. MCP Inspector
    correctly separates these into "URIs" and "Templates" sections and shows all 3.
- **Prompts** (`@mcp.prompt`) are reusable instruction templates the client can offer as
  quick actions (`sales_analysis`, `customer_segmentation`).

## Transport

`cli.py serve` defaults to **stdio**: the client launches the Python process and
communicates over its stdin/stdout using line-delimited JSON-RPC. This means **stdout
must carry nothing but protocol messages** — anything else (a stray `print()`, a log
line) corrupts the stream for a strict client.

> **Bug found and fixed during inspection:** the original `cli.py` printed three
> human-readable status lines ("Initializing database...", "Database ready.",
> "Starting MCP server on stdio") straight to stdout via `rich.console.Console()`
> *before* handing control to `mcp.run()`. FastMCP's own startup banner and logs
> correctly go to stderr — only the project's own three lines were misplaced. Fixed by
> giving those three calls their own `Console(stderr=True)` instance. Verified before/after
> by redirecting stdout/stderr to files and diffing what landed where.

`--transport http` is also supported, binding a Streamable HTTP server (used by the
`Dockerfile`, which isn't part of the original brief but ships in the repo).

## Known, documented tradeoff: `query_database`

The `query_database` tool takes one raw SQL string from the LLM and only checks that it
starts with `SELECT` (`database.py`'s `query_db()`). It is **not** parameterized in the
sense the README's wording implies — there's no bound-parameter interface for the LLM to
use, just a prefix check. The repo's own `docs/ADR.md` (decision **D7**) already documents
this as an accepted tradeoff for a portfolio/demo project, and names the fix for
production use: an AST-level SQL parser (e.g. `sqlglot`) instead of a string-prefix check.
Good, ready-made talking point for "how errors and invalid arguments are handled."

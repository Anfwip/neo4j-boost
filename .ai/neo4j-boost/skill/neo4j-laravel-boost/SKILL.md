---
name: neo4j-laravel-boost
description: "Apply this skill whenever working with the neo4j/laravel-boost package. Triggers on: setting up Neo4j MCP tools in a Laravel project, using or extending the boost:mcp server, querying the Neo4j graph schema or running Cypher queries through an AI agent, exploring the Laravel container dependency graph, contributing knowledge to the container graph, configuring transport modes (driver/stdio/http), running neo4j-boost Artisan commands, troubleshooting MCP connectivity, or adding new MCP tools to this package."
license: MIT
metadata:
  author: neo4j
---

# Neo4j Laravel Boost — AI Agent Skill

This skill teaches the AI agent how to correctly install, configure, use, and extend the `neo4j/laravel-boost` package. It provides one unified MCP server (`php artisan boost:mcp`) that exposes both Laravel Boost tools and official Neo4j graph tools to any MCP-compatible client (Cursor, Claude Code, etc.).

---

## Package Purpose

`neo4j/laravel-boost` bridges the gap between AI coding assistants and a live Neo4j database inside a Laravel project. It allows an AI agent to:

- **Inspect the live Neo4j graph schema** (labels, relationship types, property keys).
- **Run read-only and write Cypher queries** against the database.
- **Explore the Laravel container dependency graph** — both runtime DI wiring and statically-detected hidden edges.
- **Contribute new dependency or binding knowledge** back into the graph for future queries.
- **List Graph Data Science (GDS) procedures** (if the GDS plugin is installed).

All of this is exposed through a **single MCP server entry**: `php artisan boost:mcp`.

---

## Installation

```bash
# Dev dependency only — never install in production
composer require --dev neo4j/laravel-boost
```

**Requirements:** PHP 8.2+, Laravel 12 or 13.

---

## Environment Variables

Add to `.env`. The **default transport is `driver`** — no binary needed:

```env
# Required for driver/stdio/HTTP transport
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your-password

# Transport selection (default: driver)
# Options: driver | stdio | http
NEO4J_MCP_TRANSPORT=driver

# HTTP transport only
NEO4J_MCP_URL=http://localhost:8080/mcp

# Optional tuning
NEO4J_DATABASE=neo4j
NEO4J_SCHEMA_SAMPLE_SIZE=100

# Container graph static scan (comma-separated absolute paths)
NEO4J_CONTAINER_GRAPH_STATIC_SCAN_PATHS=/absolute/path/to/app/Services

# Override binary platform detection (stdio transport only)
NEO4J_MCP_PLATFORM_ASSET=
```

> **Always run `php artisan config:clear` after editing `.env`.** Laravel caches config and the change won't be picked up otherwise.

---

## MCP Client Configuration

Add exactly this entry to your MCP client. The workspace must be opened at your **Laravel application root** (where `artisan` lives):

```json
{
  "mcpServers": {
    "laravel-boost": {
      "command": "php",
      "args": ["artisan", "boost:mcp"],
      "env": {
        "APP_ENV": "local"
      }
    }
  }
}
```

**For Cursor:** place in `.cursor/mcp.json` at the project root, or generate it automatically:

```bash
php artisan neo4j-boost:cursor-config
```

**For Claude Code:** add the same entry to `claude_code_config.json` or run `claude mcp add`.

> **Critical:** `APP_ENV=local` is required. Laravel Boost commands are only registered in `local` environment. Omitting it causes "There are no commands defined in the 'boost' namespace".

---

## Transport Modes

| Mode | How it works | When to use |
|------|-------------|-------------|
| `driver` | Runs MCP tools in PHP directly via Bolt (`laudis/neo4j-php-client`). No binary. **This is the default.** | Local dev, CI, most use cases |
| `stdio` | Spawns the official `neo4j-mcp` binary as a subprocess via stdin/stdout. | When you need the exact official MCP server behavior |
| `http` | Connects to a remote or containerized MCP server over HTTP. | Docker Compose setups, shared infrastructure |

### Switching to STDIO

```env
NEO4J_MCP_TRANSPORT=stdio
```

```bash
php artisan neo4j-boost:install-mcp  # downloads the binary
```

### Switching to HTTP

```env
NEO4J_MCP_TRANSPORT=http
NEO4J_MCP_URL=http://localhost:8080/mcp
```

> In HTTP mode, the package injects `NEO4J_URI`, `NEO4J_USERNAME`, and `NEO4J_PASSWORD` with every request. Do **not** set credentials on the MCP server container itself.

---

## Available MCP Tools

These tools are automatically registered into `boost:mcp` by `Neo4jBoostServiceProvider::mergeBoostTools()`.

### `get-schema`
Returns the graph schema: labels, relationship types, property keys, and a visualization.  
**No arguments required.**  
Uses `apoc.meta.schema` if APOC is installed; falls back to `db.labels` / `db.relationshipTypes` otherwise.

### `read-cypher`
Executes a read-only Cypher query.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | ✅ | The Cypher `MATCH` / `RETURN` query |
| `params` | object | ❌ | Named query parameters |

Example:
```json
{ "query": "MATCH (n:User) RETURN n.name LIMIT 10" }
```

### `write-cypher`
Executes a write Cypher query (`CREATE`, `SET`, `DELETE`, `MERGE`, etc.).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | ✅ | The write Cypher statement |
| `params` | object | ❌ | Named query parameters |

Example:
```json
{ "query": "CREATE (u:User {name: $name})", "params": { "name": "Alice" } }
```

### `list-gds-procedures`
Lists available Graph Data Science procedures.  
**No arguments.** Requires the GDS plugin on Neo4j. If GDS is missing, this tool errors — `get-schema`, `read-cypher`, `write-cypher` still work normally.

### `get-class-dependency-graph`
Returns the Laravel container dependency graph for a PHP class. Requires `php artisan container:graph` to have been run first.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `class` | string | — | Fully-qualified class name, e.g. `App\Services\FooService` |
| `depth` | integer | `4` | Max hops (1–10) |
| `direction` | string | `outbound` | `outbound` (dependencies), `inbound` (dependents), or `both` |
| `include_bindings` | boolean | `true` | Include `BINDS_TO` edges |
| `page` | integer | `1` | Page number |
| `per_page` | integer | `100` | Results per page (max 200) |

Example:
```json
{ "class": "App\\Services\\FooService", "direction": "outbound", "depth": 4 }
```

### `contribute-graph-knowledge`
Adds a dependency or binding edge to the graph when static analysis missed it. Requires `php artisan container:graph` to have been run first.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `relationship` | string | ✅ | `DEPENDS_ON` or `BINDS_TO` |
| `from` | string | ✅ | Source FQN |
| `to` | string | ✅ | Target FQN |
| `confidence` | string | ✅ | `high` (persist immediately), `medium` or `low` (require user confirmation) |
| `confirmed` | boolean | ❌ | Set `true` after user confirms a `medium`/`low` proposal |
| `reason` | string | ❌ | Explanation for the edge |
| `shared` | boolean | ❌ | `BINDS_TO` only: is it a singleton? |
| `depends_on_type` | string | ❌ | `DEPENDS_ON` only: `constructor_injection`, `method_injection`, `facade`, `global_helper`, `service_location`, `instantiation` |

**Confidence workflow:**
- `high` → persisted immediately with `source=agent`.
- `medium`/`low` → returns `status: confirmation_required`. Ask the user, then retry with `confirmed: true` to persist with `source=user`.

---

## Artisan Commands

| Command | What it does |
|---------|-------------|
| `neo4j-boost:setup` | Interactive setup: checks connection, optionally installs binary, configures Docker Neo4j, writes Cursor config. Use `--no-interaction` for CI. |
| `neo4j-boost:start-neo4j` | Boots a local Neo4j Docker instance (`bolt://localhost:7687`, `http://localhost:7474`) with APOC. Add `--recreate` to reset. |
| `neo4j-boost:cursor-config` | Creates/updates `.cursor/mcp.json`. |
| `neo4j-boost:install-mcp` | Downloads the official `neo4j-mcp` binary (only needed for STDIO transport). |
| `neo4j-boost:doctor` | Diagnoses transport mode, binary, password, and overall readiness. Run this first when troubleshooting. |
| `neo4j-boost:test-stdio` | End-to-end verbose test of the STDIO handshake and all tools. |
| `container:graph` | Exports the Laravel container's runtime bindings into Neo4j as a graph. |

### `container:graph` flags

```bash
php artisan container:graph              # Full export
php artisan container:graph --dry-run    # Preview without writing to Neo4j
php artisan container:graph --print-cypher  # Print generated Cypher statements
```

Enable static scanning for hidden edges (service-locator calls, Facade usage, `new` instantiations):

```env
NEO4J_CONTAINER_GRAPH_STATIC_SCAN_PATHS=/absolute/path/to/app/Services,/absolute/path/to/app/Http
```

---

## Docker Compose Setup (HTTP Mode)

```yaml
services:
  neo4j:
    image: neo4j:5-community
    environment:
      NEO4J_AUTH: neo4j/your-password
      NEO4J_PLUGINS: '["apoc", "graph-data-science"]'
      NEO4J_dbms_security_procedures_unrestricted: 'apoc.*,gds.*'
      NEO4J_dbms_security_procedures_allowlist: 'apoc.*,gds.*'
    ports:
      - "7474:7474"
      - "7687:7687"
    healthcheck:
      test: ["CMD-SHELL", "wget -q -O /dev/null http://localhost:7474 || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s

  neo4j-mcp:
    image: mcp/neo4j
    environment:
      NEO4J_URI: bolt://neo4j:7687
      NEO4J_DATABASE: neo4j
      NEO4J_READ_ONLY: "false"
      NEO4J_TELEMETRY: "false"
      NEO4J_TRANSPORT_MODE: http
      NEO4J_MCP_HTTP_HOST: 0.0.0.0
      NEO4J_MCP_HTTP_PORT: "8080"
    ports:
      - "8080:8080"
    depends_on:
      neo4j:
        condition: service_healthy
```

Laravel `.env` for this setup:

```env
NEO4J_MCP_TRANSPORT=http
NEO4J_MCP_URL=http://localhost:8080/mcp
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your-password
```

---

## Automating Setup After Dependency Updates

Add to `composer.json` so config is always re-published after `composer update`:

```json
{
  "scripts": {
    "post-update-cmd": [
      "@php artisan neo4j-boost:setup --no-interaction"
    ]
  }
}
```

---

## Publishing Config (Optional)

```bash
php artisan vendor:publish --tag=neo4j-boost-config
```

This copies `config/neo4j-boost.php` into the project. Useful for tweaking `bolt.schema_sample_size`, `container_graph.static_scan_paths`, etc. directly.

---

## How the Package Works Internally

- **`Neo4jBoostServiceProvider`** merges Neo4j tools into `boost.mcp.tools.include` at boot, so `php artisan boost:mcp` automatically picks them up alongside Laravel Boost tools.
- **Transport resolution** happens in the service provider: `NEO4J_MCP_TRANSPORT` determines whether `Neo4jDriverClient` (Bolt/PHP), `Neo4jStdioClient` (subprocess), or `Neo4jHttpClient` (HTTP) is bound to `Neo4jMcpClientInterface`.
- **`DevDependencyConfigPublisher`** auto-publishes `config/neo4j-boost.php` on first boot so developers don't need to run `vendor:publish` manually.
- All tools implement `Laravel\Mcp\Server\Tool` and validate their inputs with Laravel's `Validator` facade before calling the underlying client.

---

## Common Issues & Troubleshooting

| Symptom | Fix |
|---------|-----|
| `"Could not open input file: artisan"` | Open the **Laravel application folder** as workspace, not the package directory. |
| `"There are no commands defined in the 'boost' namespace"` | Add `"APP_ENV": "local"` to the MCP server env config. |
| `"Neo4j password is required"` (STDIO) | Set `NEO4J_PASSWORD` in `.env`, then run `php artisan config:clear`. |
| APOC/meta errors | Neo4j instance missing APOC. Run `php artisan neo4j-boost:start-neo4j --recreate`. |
| `bolt://localhost:7687` unreachable in Docker | Use the service hostname: `bolt://neo4j:7687`. Run `php artisan config:clear`. |
| HTTP 404 `"This server only handles requests to /mcp"` | Ensure `NEO4J_TRANSPORT_MODE=http` on the server and the URL ends with `/mcp`. Prefer `boost:mcp` as the client endpoint. |
| `"Unknown function 'gds.version'"` | GDS plugin missing. Schema and Cypher tools still work. See Docker setup for plugin config. |
| Changes to `.env` not reflected | Always run `php artisan config:clear` after editing `.env`. |

---

## How to Apply This Skill

1. **Setting up the package?** → Follow Installation → Environment Variables → MCP Client Configuration in order.
2. **Choosing transport?** → Default to `driver`. Only switch to `stdio` or `http` if there's a specific reason.
3. **Agent wants to query the database?** → Use `get-schema` first to understand node/relationship types, then `read-cypher`.
4. **Agent wants to understand Laravel DI wiring?** → Run `container:graph` to export, then use `get-class-dependency-graph`.
5. **Agent discovers a hidden dependency?** → Use `contribute-graph-knowledge` with appropriate confidence level.
6. **Something broken?** → Run `php artisan neo4j-boost:doctor` and check the troubleshooting table above.

# RDS Supabase MCP

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![npm](https://img.shields.io/npm/v/rds-supabase-mcp)](https://www.npmjs.com/package/rds-supabase-mcp)
[![Node](https://img.shields.io/badge/node-%3E%3D18-brightgreen)](https://nodejs.org)

**[中文](README.md) · [English](README.en.md)**

A full-featured management MCP (Model Context Protocol) server for cloud-hosted instances without direct database access.

> **Secondary Development Notice & Acknowledgements**
>
> This project is a secondary development based on the open-source project [HenkDz/selfhosted-supabase-mcp](https://github.com/HenkDz/selfhosted-supabase-mcp) (MIT License); the code baseline is upstream main branch commit `db8d846`.
>
> Special thanks to the original author [HenkDz](https://github.com/HenkDz), as well as [nightness](https://github.com/nightness), [calclavia](https://github.com/calclavia), [dalisys](https://github.com/dalisys), [moatus](https://github.com/moatus), and all upstream contributors — this project exists thanks to their open-source dedication. Licensing and copyright notices are in [LICENSE](LICENSE) and [NOTICE](NOTICE).

## 1. Background

Cloud-hosted Supabase instances expose only an HTTP API gateway for security and operational reasons; the PostgreSQL direct-connection ports are not open to the public. As a result, neither of the two existing approaches applies:

- **Generic Postgres MCP servers** depend on a direct database connection and are entirely unusable when the port is unreachable.
- **The official Supabase cloud MCP server** is bound to the platform's Management API and does not fit self-hosted or gateway-based hosted deployments.

This project closes that gap: **every database operation is performed via the PostgREST interface behind the gateway over RPC, with no direct database connection required at any point.**

## 2. Key Features

- **No direct connection required**: full management capability with just the gateway URL and two keys (`--url` / `--anon-key` / `--service-key`)
- **43 management tools**: SQL execution, schema inspection, RLS and security advisors, statistics and logs, auth users, storage, scheduled jobs, Edge Functions
- **Privilege tiers**: tools are classified as Regular or Privileged; privileged tools are restricted to the service role
- **Two transports**: stdio (local IDE clients) and HTTP (remote deployment with JWT authentication, role-based access control, rate limiting, and CORS)
- **Statement-level read-only gate**: when `execute_sql` is called with `read_only: true`, only query-type statements are allowed and rejected before reaching the execution engine

## 3. Tool Overview

| Category | Tools | Privilege |
|---|---|---|
| SQL & execution plans | `execute_sql`, `explain_query` | Privileged |
| Schema inspection | `list_tables`, `list_table_columns`, `list_indexes`, `list_constraints`, `list_foreign_keys`, `list_triggers`, `list_database_functions`, `get_function_definition`, `get_trigger_definition` | Regular |
| Extensions & migrations | `list_extensions`, `list_available_extensions`, `list_migrations`, `apply_migration` | Regular / Privileged |
| Statistics | `get_database_connections`, `get_database_stats`, `get_index_stats`, `get_vector_index_stats`, `list_vector_indexes` | Regular |
| Security & RLS | `list_rls_policies`, `get_rls_status`, `get_advisors` | Regular |
| Auth users | `list_auth_users`, `get_auth_user`, `create_auth_user`, `update_auth_user`, `delete_auth_user` | Regular / Privileged |
| Storage | `get_storage_config`, `update_storage_config`, `list_storage_buckets`, `list_storage_objects` | Regular / Privileged |
| Realtime | `list_realtime_publications` | Regular |
| Scheduled jobs | `list_cron_jobs`, `get_cron_job_history` | Regular |
| Edge Functions | `list_edge_functions`, `get_edge_function_details`, `list_edge_function_logs` | Regular |
| Development aids | `generate_typescript_types`, `get_logs`, `rebuild_hooks`, `get_project_url`, `verify_jwt_secret` | Regular / Privileged |

## 4. Quick Start

### Run via npx (recommended)

npm package `rds-supabase-mcp` (Node.js >= 18):

```bash
npx -y rds-supabase-mcp \
  --url <gateway-url> \
  --anon-key <anon-key> \
  --service-key <service-role-key>
```

### Build from source

Prerequisite: [Bun](https://bun.sh/) v1.1 or later.

```bash
git clone <repository-url>
cd rds-supabase-mcp
bun install
bun run build    # produces dist/ (Node-compatible; runs under node and bun)
```

Direct-connection mode (optional): add `--db-url postgresql://user:pass@host:5432/db` to unlock migration execution and the direct-connection auth/storage tools (see Section 7).

## 5. Configuration Reference

| Argument | Environment variable | Required | Description |
|---|---|---|---|
| `--url` | `SUPABASE_URL` | Yes | Instance gateway URL (Kong entry point) |
| `--anon-key` | `SUPABASE_ANON_KEY` | Yes | Anonymous key |
| `--service-key` | `SUPABASE_SERVICE_ROLE_KEY` | Recommended | Service role key; required for privileged tools and the `execute_sql` RPC |
| `--db-url` | `DATABASE_URL` | No | PostgreSQL connection string; required by direct-connection tools |
| `--jwt-secret` | `SUPABASE_AUTH_JWT_SECRET` | Required in HTTP mode | JWT verification secret |
| `--tools-config` | — | No | Tool whitelist JSON: `{"enabledTools": ["tool_name"]}` |

Additional HTTP transport arguments: `--transport http`, `--port` (default 3000), `--host` (default 127.0.0.1), `--cors-origins` (localhost only by default), `--rate-limit-window` (default 60000ms), `--rate-limit-max` (default 100), `--request-timeout` (default 30000ms).

## 6. Client Configuration (stdio)

Example configuration for a generic MCP client:

```json
{
  "mcpServers": {
    "rds-supabase": {
      "command": "npx",
      "args": [
        "-y", "rds-supabase-mcp",
        "--url", "https://<gateway-url>",
        "--anon-key", "<anon-key>",
        "--service-key", "<service-role-key>"
      ]
    }
  }
}
```

## 7. No-Direct-Connection Mode

When `--db-url` is not configured:

- **Available**: SQL execution, schema inspection, statistics, RLS and advisors, logs, scheduled jobs, Edge Functions, type generation, and most other tools — all performed over PostgREST RPC
- **Unavailable (requires a direct connection)**: `apply_migration`, the five auth user tools, `list_storage_buckets`, `list_storage_objects`, `list_realtime_publications`
- **Prerequisite**: the `public.execute_sql` function must exist in the database. The server creates it automatically at startup if it is missing and both a service key and a database URL are provided; in no-connection environments, deploy it beforehand via the management plane (e.g. the postgres-meta service behind Kong's `/pg/query` route on self-hosted stacks)
- **Read-only gate**: with `read_only: true`, only query-type statements are allowed (select / with / values / table / explain; data-modifying CTEs and EXPLAIN ANALYZE are rejected)

## 8. Security Model

**stdio mode**: no authentication; all tools are callable. Intended only for trusted local clients (e.g. IDEs). A warning is printed at startup.

**HTTP mode** (recommended for remote access):

- JWT verification is enforced on all `/mcp` endpoints
- Role-based access control driven by the JWT `role` claim: `service_role` gets all tools; `authenticated` gets regular tools only; `anon` is denied
- Per-IP rate limiting, CORS allow-list, security headers, request timeouts

**Password handling**: passwords for the auth user tools are hashed with bcrypt via pgcrypto before storage; queries are parameterized throughout and plaintext passwords are never persisted.

**SQL injection protection**: all management tools use parameterized queries. `execute_sql` is an intentional arbitrary-SQL exception by design, with its exposure contained by the service role privilege tier.

## 9. Differences from Upstream

- Packaging: npm package name `rds-supabase-mcp`; added `bin` / `files` / `engines` metadata; build target switched from Bun to Node (with a shebang injected at the entry) so the output runs under `node`, `npx`, and `bun`; CLI and MCP server identity renamed to this package (`rds-supabase-mcp@0.1.0`); removed the upstream Smithery publishing configuration
- Deployment side: hardened `execute_sql` database function v4.1 adapted for no-connection environments — fixes the DDL channel defect in the upstream v1 definition (`42P11` / `42P01` root-cause chain) and adds a statement-level read-only gate with case-insensitive keyword dispatch

## 10. Acknowledgements

This project is a secondary development based on the open-source project [HenkDz/selfhosted-supabase-mcp](https://github.com/HenkDz/selfhosted-supabase-mcp) (MIT License). The initial code baseline is upstream main branch commit `db8d846`. Thanks to the original author [HenkDz](https://github.com/HenkDz) and all contributors for the open-source contribution.

## 11. License

This project and the upstream code it contains are both licensed under the [MIT License](LICENSE); copyright notices are in the LICENSE file. Upstream provenance and attribution are additionally documented in [NOTICE](NOTICE) and Section 10. Please retain the copyright and license notices from LICENSE when redistributing.

## 12. Trademark Notice

Supabase is a trademark of Supabase, Inc. This project is not affiliated with or endorsed by Supabase, Inc.; the name is used for compatibility description only.

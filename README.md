# RDS Supabase MCP

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![npm](https://img.shields.io/npm/v/rds-supabase-mcp)](https://www.npmjs.com/package/rds-supabase-mcp)
[![Node](https://img.shields.io/badge/node-%3E%3D18-brightgreen)](https://nodejs.org)

**[中文](README.md) · [English](README.en.md)**

面向云上托管 / 无直连实例的全功能管理 MCP（Model Context Protocol）服务器。

> **二次开发声明与致谢**
>
> 本项目基于开源项目 [HenkDz/selfhosted-supabase-mcp](https://github.com/HenkDz/selfhosted-supabase-mcp)（MIT License）二次开发而成，代码基线为上游 main 分支 `db8d846`。
>
> 特别感谢原作者 [HenkDz](https://github.com/HenkDz)，以及 [nightness](https://github.com/nightness)、[calclavia](https://github.com/calclavia)、[dalisys](https://github.com/dalisys)、[moatus](https://github.com/moatus) 等全体上游贡献者——正是他们的开源奉献，本项目才得以存在。许可与版权声明见 [LICENSE](LICENSE) 与 [NOTICE](NOTICE)。

## 1. 项目背景

云上托管的 Supabase 实例出于安全与运维考虑，通常只对外暴露 HTTP API 网关，不开放 PostgreSQL 直连端口。这使得两类现有方案均不适用：

- **通用 Postgres MCP**：依赖数据库直连，端口不通即完全不可用；
- **Supabase 官方云版 MCP**：绑定官方云平台 Management API，不适用于自托管或托管网关形态。

本项目针对该场景提供完整方案：**全部数据库操作经网关内的 PostgREST 接口以 RPC 方式完成，自始至终不需要直连数据库。**

## 2. 核心特性

- **无直连架构**：仅需网关地址与两个密钥（`--url` / `--anon-key` / `--service-key`）即可获得全功能管理能力
- **43 个管理工具**：覆盖 SQL 执行、结构查看、RLS 与安全体检、统计与日志、Auth 用户、存储、定时任务、Edge Functions
- **权限分级**：工具分为常规（Regular）与特权（Privileged）两级，特权工具仅限 service role 调用
- **双传输模式**：stdio（本地 IDE 客户端）与 HTTP（远程部署，含 JWT 鉴权、角色控制、限流、CORS）
- **语句级只读门禁**：`execute_sql` 传 `read_only: true` 时，仅放行查询类语句，在进入执行引擎前拦截

## 3. 工具总览

| 分类 | 工具 | 权限 |
|---|---|---|
| SQL 与执行计划 | `execute_sql`、`explain_query` | 特权 |
| 模式查看 | `list_tables`、`list_table_columns`、`list_indexes`、`list_constraints`、`list_foreign_keys`、`list_triggers`、`list_database_functions`、`get_function_definition`、`get_trigger_definition` | 常规 |
| 扩展与迁移 | `list_extensions`、`list_available_extensions`、`list_migrations`、`apply_migration` | 常规 / 特权 |
| 统计 | `get_database_connections`、`get_database_stats`、`get_index_stats`、`get_vector_index_stats`、`list_vector_indexes` | 常规 |
| 安全与 RLS | `list_rls_policies`、`get_rls_status`、`get_advisors` | 常规 |
| Auth 用户 | `list_auth_users`、`get_auth_user`、`create_auth_user`、`update_auth_user`、`delete_auth_user` | 常规 / 特权 |
| 存储 | `get_storage_config`、`update_storage_config`、`list_storage_buckets`、`list_storage_objects` | 常规 / 特权 |
| 实时 | `list_realtime_publications` | 常规 |
| 定时任务 | `list_cron_jobs`、`get_cron_job_history` | 常规 |
| Edge Functions | `list_edge_functions`、`get_edge_function_details`、`list_edge_function_logs` | 常规 |
| 开发辅助 | `generate_typescript_types`、`get_logs`、`rebuild_hooks`、`get_project_url`、`verify_jwt_secret` | 常规 / 特权 |

## 4. 快速开始

### npx 免安装运行（推荐）

npm 包 `rds-supabase-mcp`（Node.js >= 18）：

```bash
npx -y rds-supabase-mcp \
  --url <实例网关地址> \
  --anon-key <anon-key> \
  --service-key <service-role-key>
```

### 从源码构建

前置条件：[Bun](https://bun.sh/) v1.1 及以上。

```bash
git clone <repository-url>
cd rds-supabase-mcp
bun install
bun run build    # 产出 dist/（Node 兼容，node / bun 均可运行）
```

直连增强模式（可选）：在上述命令基础上追加 `--db-url postgresql://user:pass@host:5432/db`，解锁迁移执行与 Auth / 存储直连类工具（见第 7 节）。

## 5. 配置参考

| 参数 | 环境变量 | 必填 | 说明 |
|---|---|---|---|
| `--url` | `SUPABASE_URL` | 是 | 实例网关地址（Kong 入口） |
| `--anon-key` | `SUPABASE_ANON_KEY` | 是 | 匿名密钥 |
| `--service-key` | `SUPABASE_SERVICE_ROLE_KEY` | 建议 | service role 密钥；特权工具与 `execute_sql` RPC 所需 |
| `--db-url` | `DATABASE_URL` | 否 | PostgreSQL 直连串；直连类工具所需 |
| `--jwt-secret` | `SUPABASE_AUTH_JWT_SECRET` | HTTP 模式必填 | JWT 校验密钥 |
| `--tools-config` | — | 否 | 工具白名单 JSON：`{"enabledTools": ["tool_name"]}` |

HTTP 传输模式追加参数：`--transport http`、`--port`（默认 3000）、`--host`（默认 127.0.0.1）、`--cors-origins`（默认仅 localhost）、`--rate-limit-window`（默认 60000ms）、`--rate-limit-max`（默认 100）、`--request-timeout`（默认 30000ms）。

## 6. 客户端接入（stdio）

通用 MCP 客户端配置示例：

```json
{
  "mcpServers": {
    "rds-supabase": {
      "command": "npx",
      "args": [
        "-y", "rds-supabase-mcp",
        "--url", "https://<实例网关地址>",
        "--anon-key", "<anon-key>",
        "--service-key", "<service-role-key>"
      ]
    }
  }
}
```

## 7. 无直连模式说明

不配置 `--db-url` 时：

- **可用**：SQL 执行、结构查看、统计、RLS 与顾问、日志、定时任务、Edge Functions、类型生成等绝大多数工具，均经 PostgREST RPC 完成
- **不可用（需直连）**：`apply_migration`、Auth 用户管理五件套、`list_storage_buckets`、`list_storage_objects`、`list_realtime_publications`
- **依赖前提**：数据库中需存在 `public.execute_sql` 函数。若启动时同时提供 service-key 与 db-url 且函数缺失，服务器将自动创建；无直连环境请经管理面（如自托管栈中 Kong `/pg/query` 指向的 postgres-meta 服务）预先部署
- **只读门禁**：`read_only: true` 时仅放行查询类语句（select / with / values / table / explain，且禁止数据修改 CTE 与 EXPLAIN ANALYZE）

## 8. 安全模型

**stdio 模式**：无鉴权，全部工具可调用，仅适用于本地可信客户端（IDE 等），启动时会打印警告。

**HTTP 模式**（推荐用于远程访问）：

- 全部 `/mcp` 端点强制 JWT 校验
- 基于 JWT `role` 声明的访问控制：`service_role` 全量工具；`authenticated` 仅常规工具；`anon` 拒绝访问
- 每 IP 限流、CORS 白名单、安全响应头、请求超时

**密码处理**：Auth 用户工具的密码经 pgcrypto bcrypt 哈希后落库，全程参数化查询，明文不落盘。

**SQL 注入防护**：所有管理工具使用参数化查询；`execute_sql` 是设计上的任意 SQL 例外，已通过 service role 特权级收敛暴露面。

## 9. 与上游的差异

- 发布工程：npm 包名 `rds-supabase-mcp`；新增 `bin` / `files` / `engines` 元数据；构建目标由 Bun 改为 Node（入口注入 shebang），`node` / `npx` / `bun` 均可运行；CLI 与 MCP Server 标识改为本包名（`rds-supabase-mcp@0.1.0`）；移除上游的 Smithery 发布配置
- 部署侧：适配无直连场景的 `execute_sql` 数据库函数加固版 v4.1 —— 修复上游 v1 定义的 DDL 通道缺陷（`42P11` / `42P01` 根因链），增加语句级 `read_only` 门禁与大小写无关的关键词分发

## 10. 来源与致谢

本项目基于开源项目 [HenkDz/selfhosted-supabase-mcp](https://github.com/HenkDz/selfhosted-supabase-mcp)（MIT License）二次开发，首个代码基线为上游 main 分支 `db8d846`。感谢原作者 [HenkDz](https://github.com/HenkDz) 及全体贡献者的开源贡献。

## 11. 许可证

本项目与所包含的上游代码均采用 [MIT License](LICENSE)，版权声明见 LICENSE 文件；上游来源与署名另见 [NOTICE](NOTICE) 与第 10 节。再分发时请保留 LICENSE 中的版权与许可声明。

## 12. 商标声明

Supabase 是 Supabase, Inc. 的商标；本项目与其无隶属或背书关系，仅作兼容性描述。

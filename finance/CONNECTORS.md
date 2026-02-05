# Connectors

## How tool references work

Plugin files use `~~category` as a placeholder for whatever tool the user connects in that category. For example, `~~data warehouse` might mean Snowflake, BigQuery, or any other warehouse with an MCP server.

Plugins are **tool-agnostic** — they describe workflows in terms of categories (data warehouse, chat, project tracker, etc.) rather than specific products. The `.mcp.json` pre-configures specific MCP servers, but any MCP server in that category works.

## Connectors for this plugin

| Category | Placeholder | Included servers | Other options |
|----------|-------------|-----------------|---------------|
| Local database | `~~database` | **DuckDB** (primary) | PostgreSQL, SQLite |
| Data warehouse | `~~data warehouse` | Snowflake\*, Databricks\*, BigQuery | Redshift, PostgreSQL |
| Email | `~~email` | Microsoft 365 | — |
| Office suite | `~~office suite` | Microsoft 365 | — |
| Chat | `~~chat` | Slack | Microsoft Teams |
| ERP / Accounting | `~~erp` | — (no supported MCP servers yet) | NetSuite, SAP, QuickBooks, Xero |
| Analytics / BI | `~~analytics` | — (no supported MCP servers yet) | Tableau, Looker, Power BI |

### DuckDB - Primary Database

DuckDB is configured as the primary analytical database for this plugin with support for:
- **Parquet files**: Efficient columnar storage for financial data
- **VSS extension**: Semantic search on financial documents and transactions
- **JSON extension**: Flexible data structures for complex financial records
- **FTS extension**: Full-text search on descriptions and notes
- **HTTPFS extension**: Direct access to cloud storage (S3, GCS, Azure)

See [DUCKDB.md](./DUCKDB.md) for detailed usage examples and configuration.

\* Placeholder — MCP URL not yet configured

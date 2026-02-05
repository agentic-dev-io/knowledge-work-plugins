# DuckDB Integration

All knowledge work plugins now include **DuckDB** integration for high-performance analytics and data management. DuckDB is a fast, embedded analytical database that runs locally on your machine without requiring a separate server.

## Why DuckDB?

DuckDB is ideal for knowledge work plugins because it provides:

1. **Zero Configuration**: No server setup required - it's embedded directly in your workflow
2. **High Performance**: Columnar storage and vectorized execution for fast analytics
3. **Rich Extensions**: Vector similarity search (VSS), full-text search (FTS), JSON handling, and more
4. **Multiple Formats**: Native support for Parquet, CSV, JSON, and direct cloud storage access
5. **SQL Standard**: Full SQL support with window functions, CTEs, and advanced analytics
6. **Portable**: Database files are single files that can be easily backed up or shared

## Key Extensions Used Across Plugins

### 1. VSS (Vector Similarity Search)
Find semantically similar content using embeddings.

**Use Cases:**
- Finance: Similar transactions and financial documents
- Bio-Research: Similar research papers and protein sequences
- Customer Support: Similar support tickets for solution suggestions
- Sales: Comparable deals and opportunities
- Marketing: Similar campaigns and audience segments

### 2. FTS (Full-Text Search)
Fast keyword-based search with BM25 ranking.

**Use Cases:**
- Enterprise Search: Search across all company content
- Legal: Find specific contract clauses
- Product Management: Search feature requests
- All plugins: Search notes, descriptions, and documentation

### 3. Parquet Support
Efficient columnar storage for large datasets.

**Use Cases:**
- Finance: Financial transactions and reports
- Bio-Research: Genomics data and experimental results
- Data: Analytics datasets and data warehouse exports
- Product Management: Usage events and metrics

### 4. JSON Extension
Handle semi-structured data flexibly.

**Use Cases:**
- Sales: Account hierarchies and relationships
- Legal: Contract metadata and clause structures
- All plugins: API responses and flexible schemas

### 5. HTTPFS Extension
Direct access to cloud storage (S3, GCS, Azure).

**Use Cases:**
- All plugins: Access data lakes without downloading
- Data: Query data warehouses directly
- Bio-Research: Access public genomics datasets

## Database Locations

Each plugin stores its DuckDB database in a dedicated directory:

```
~/knowledge-work-data/
├── finance/finance.duckdb
├── bio-research/bio-research.duckdb
├── product-management/product-management.duckdb
├── customer-support/customer-support.duckdb
├── enterprise-search/enterprise-search.duckdb
├── sales/sales.duckdb
├── marketing/marketing.duckdb
├── productivity/productivity.duckdb
├── legal/legal.duckdb
└── data/data.duckdb
```

## Quick Start Examples

### Loading Data
```sql
-- From Parquet
CREATE TABLE my_data AS SELECT * FROM 'data.parquet';

-- From CSV
CREATE TABLE my_data AS SELECT * FROM 'data.csv';

-- From JSON
CREATE TABLE my_data AS SELECT * FROM 'data.json';

-- From S3
CREATE TABLE my_data AS SELECT * FROM 's3://bucket/data/*.parquet';
```

### Basic Analytics
```sql
-- Aggregations
SELECT category, COUNT(*), SUM(amount), AVG(amount)
FROM transactions
GROUP BY category;

-- Time series
SELECT DATE_TRUNC('month', date) as month, SUM(revenue)
FROM sales
GROUP BY month
ORDER BY month;

-- Window functions
SELECT *, 
       ROW_NUMBER() OVER (PARTITION BY category ORDER BY amount DESC) as rank,
       AVG(amount) OVER (PARTITION BY category) as category_avg
FROM transactions;
```

### Vector Similarity Search
```sql
-- Install and load VSS extension
INSTALL vss;
LOAD vss;

-- Create table with embeddings
CREATE TABLE documents (
    id INT,
    text VARCHAR,
    embedding FLOAT[384]
);

-- Create HNSW index
CREATE INDEX docs_idx ON documents 
USING HNSW (embedding) WITH (metric = 'cosine');

-- Find similar documents
SELECT id, text,
       array_cosine_similarity(embedding, $query_embedding::FLOAT[384]) as similarity
FROM documents
ORDER BY similarity DESC
LIMIT 10;
```

### Full-Text Search
```sql
-- Install and load FTS extension
INSTALL fts;
LOAD fts;

-- Create full-text index
PRAGMA create_fts_index('documents', 'id', 'title', 'content');

-- Search
SELECT * FROM (
    SELECT *, fts_main_documents.match_bm25(id, 'search query') AS score
    FROM documents
) WHERE score IS NOT NULL
ORDER BY score DESC;
```

## Plugin-Specific Documentation

Each plugin has detailed DuckDB documentation with use cases tailored to that domain:

- [Finance Plugin](./finance/DUCKDB.md) - Financial analytics, reconciliation, variance analysis
- [Bio-Research Plugin](./bio-research/DUCKDB.md) - Genomics analysis, research literature
- [Product Management Plugin](./product-management/DUCKDB.md) - Metrics tracking, feature adoption
- [Customer Support Plugin](./customer-support/DUCKDB.md) - Ticket analytics, knowledge base
- [Enterprise Search Plugin](./enterprise-search/DUCKDB.md) - Semantic search across content
- [Sales Plugin](./sales/DUCKDB.md) - Pipeline analytics, forecasting
- [Marketing Plugin](./marketing/DUCKDB.md) - Campaign analytics, attribution
- [Productivity Plugin](./productivity/DUCKDB.md) - Task tracking, calendar analysis
- [Legal Plugin](./legal/DUCKDB.md) - Contract analysis, document management
- [Data Plugin](./data/DUCKDB.md) - Comprehensive data analytics guide

## Extension Priority Matrix

Extensions are prioritized based on impact and use case alignment:

### Priority P0 (Critical) - Data & Finance Plugins

| Extension | Plugins | Purpose |
|-----------|---------|---------|
| **snowflake** | data, finance | Direct connection to Snowflake data warehouse |
| **bigquery** | data, finance | Connect to Google BigQuery |
| **databricks** | data, finance | Connect to Databricks lakehouse |
| **jsonata** | data, finance, product-management | Transform JSON data from APIs |
| **anofox_statistics** | data, finance | Regression analysis & statistical tests |
| **anofox_forecast** | data, finance | Time series forecasting |
| **faiss** | data | High-performance vector similarity search |
| **quackformers** | data | Transformer models & NLP in DuckDB |
| **pivot_table** | data, finance, product-management | Dynamic pivots & cross-tabs |
| **parser_tools** | data | SQL query parsing & optimization |
| **dash** | data | Interactive dashboard components |
| **miniplot** | data | ASCII/Unicode plots in results |

### Priority P1 (High) - Sales, Support, Search Plugins

| Extension | Plugins | Purpose |
|-----------|---------|---------|
| **rapidfuzz** | sales, customer-support, enterprise-search, marketing, legal | Fuzzy string matching & similarity |
| **fuzzycomplete** | sales, customer-support, enterprise-search | Auto-complete with fuzzy matching |
| **webbed** | sales, marketing, product-management, enterprise-search | XML/HTML parsing & web scraping |
| **markdown** | marketing, customer-support, product-management, enterprise-search, legal | Markdown processing & generation |
| **marisa** | sales, enterprise-search, bio-research | Fast trie-based string lookups |
| **netquack** | sales | URI/Domain/IP parsing |
| **splink_udfs** | customer-support | Probabilistic record linkage |
| **tsid** | enterprise-search | Unique time-sorted IDs |

### Priority P2 (Medium) - Workflow & Analytics

| Extension | Plugins | Purpose |
|-----------|---------|---------|
| **yaml** | product-management, legal, productivity | Parse YAML configuration files |
| **datasketches** | product-management, productivity | Approximate distinct counts (memory efficient) |
| **minijinja / tera** | marketing | Text templating engines |
| **encoding** | legal | Handle different text encodings |
| **cache_httpfs** | finance, bio-research | Cached cloud storage access |

### Priority P3 (Nice-to-have) - Scientific & Specialized

| Extension | Plugins | Purpose |
|-----------|---------|---------|
| **arrow** | bio-research | Apache Arrow for large datasets |
| **hdf5** | bio-research | Scientific HDF5 data files |
| **read_stat** | bio-research | Read SAS/Stata/SPSS files |

## Installation Priority by Plugin

### Start Here (Highest Impact):

**1. DATA Plugin** - Install first for maximum analytics capability
```sql
INSTALL snowflake, bigquery, databricks;
INSTALL anofox_statistics, anofox_forecast;
INSTALL faiss, quackformers;
INSTALL pivot_table, parser_tools;
INSTALL dash, miniplot;
```

**2. FINANCE Plugin** - Critical for financial workflows
```sql
INSTALL snowflake, bigquery, databricks;
INSTALL anofox_statistics, anofox_forecast;
INSTALL cache_httpfs, pivot_table;
```

### Next Priority (High Value):

**3. SALES Plugin**
```sql
INSTALL webbed, netquack, jsonata;
INSTALL fuzzycomplete, marisa;
```

**4. CUSTOMER-SUPPORT Plugin**
```sql
INSTALL rapidfuzz, splink_udfs;
INSTALL markdown, jsonata, fuzzycomplete;
```

**5. ENTERPRISE-SEARCH Plugin**
```sql
INSTALL rapidfuzz, fuzzycomplete, marisa;
INSTALL webbed, markdown, tsid;
```

### Follow-Up (Domain-Specific):

**6. MARKETING Plugin**
```sql
INSTALL webbed, markdown, jsonata;
INSTALL anofox_statistics, rapidfuzz;
INSTALL minijinja;  -- or tera
```

**7. PRODUCT-MANAGEMENT Plugin**
```sql
INSTALL webbed, markdown, yaml, jsonata;
INSTALL datasketches, pivot_table;
```

**8. BIO-RESEARCH Plugin**
```sql
INSTALL arrow, hdf5, read_stat;
INSTALL anofox_statistics, marisa;
INSTALL cache_httpfs;
```

**9. LEGAL Plugin**
```sql
INSTALL markdown, yaml, rapidfuzz;
INSTALL encoding;
```

**10. PRODUCTIVITY Plugin**
```sql
INSTALL jsonata, yaml, datasketches;
```

## Configuration

DuckDB is configured in each plugin's `.mcp.json` file using the MCP server:

```json
{
  "mcpServers": {
    "duckdb": {
      "command": "uvx",
      "args": ["mcp-server-duckdb", "--db-path", "~/knowledge-work-data/[plugin]/[plugin].duckdb"]
    }
  }
}
```

The MCP server provides a standardized interface for Claude to interact with DuckDB databases.

## Resources

- [DuckDB Official Documentation](https://duckdb.org/docs/)
- [VSS Extension Documentation](https://duckdb.org/docs/extensions/vss)
- [Full-Text Search Documentation](https://duckdb.org/docs/extensions/full_text_search)
- [Parquet Files Documentation](https://duckdb.org/docs/data/parquet/overview)
- [MCP Server for DuckDB](https://github.com/ktanaka101/mcp-server-duckdb)

## Performance Tips

1. **Use Parquet for large datasets** - 10-100x compression and faster queries
2. **Create indexes** on frequently queried columns
3. **Use COPY for bulk loading** - Much faster than individual INSERTs
4. **Leverage window functions** - More efficient than self-joins
5. **Partition by date** - Improves query performance on time-series data
6. **Use columnar projections** - Only read the columns you need

## Next Steps

1. Review the plugin-specific documentation for your use case
2. Install extensions you need (`INSTALL extension_name; LOAD extension_name;`)
3. Load your data (Parquet, CSV, JSON, or cloud storage)
4. Create appropriate indexes for your queries
5. Start analyzing!

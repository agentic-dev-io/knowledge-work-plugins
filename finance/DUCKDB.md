# DuckDB Configuration for Finance Plugin

## Overview

This plugin uses DuckDB as its primary analytical database for financial data processing, reporting, and analysis. DuckDB provides high-performance SQL analytics with support for various data formats and extensions.

## Database Location

The DuckDB database file is stored at:
```
~/knowledge-work-data/finance/finance.duckdb
```

## Key Extensions

### 1. Parquet Extension
DuckDB has native support for Parquet files, enabling efficient columnar storage and fast analytical queries.

**Usage:**
```sql
-- Load financial data from Parquet
CREATE TABLE transactions AS 
SELECT * FROM 'financial_data.parquet';

-- Export to Parquet
COPY transactions TO 'output.parquet' (FORMAT PARQUET);
```

### 2. JSON Extension
Handle JSON data for flexible financial data structures.

**Usage:**
```sql
INSTALL json;
LOAD json;

-- Parse JSON data
SELECT json_extract(data, '$.amount') as amount 
FROM transactions;

-- Create JSON objects
SELECT json_object('account', account_id, 'balance', balance) 
FROM accounts;
```

### 3. VSS (Vector Similarity Search) Extension
Perform semantic search on financial documents, reports, and similar transactions.

**Installation:**
```sql
INSTALL vss;
LOAD vss;
```

**Usage:**
```sql
-- Create table with embeddings
CREATE TABLE financial_docs (
    id INT,
    document_text VARCHAR,
    embedding FLOAT[384]
);

-- Create HNSW index for fast similarity search
CREATE INDEX docs_embedding_idx ON financial_docs 
USING HNSW (embedding) WITH (metric = 'cosine');

-- Find similar documents
SELECT id, document_text,
       array_cosine_similarity(embedding, $1::FLOAT[384]) as similarity
FROM financial_docs
ORDER BY similarity DESC
LIMIT 10;
```

### 4. FTS (Full-Text Search) Extension
Search through financial records, notes, and descriptions.

**Installation:**
```sql
INSTALL fts;
LOAD fts;
```

**Usage:**
```sql
-- Create full-text search index
PRAGMA create_fts_index('transactions', 'id', 'description', 'notes');

-- Search
SELECT * FROM (
    SELECT *, fts_main_transactions.match_bm25(id, 'invoice payment') AS score
    FROM transactions
) WHERE score IS NOT NULL
ORDER BY score DESC;
```

### 5. HTTPFS Extension
Access data from cloud storage (S3, GCS, Azure) directly.

**Installation:**
```sql
INSTALL httpfs;
LOAD httpfs;

-- Configure for S3
SET s3_region='us-east-1';
SET s3_access_key_id='your-key';
SET s3_secret_access_key='your-secret';
```

**Usage:**
```sql
-- Query data directly from S3
SELECT * FROM 's3://bucket/financial_reports.parquet';

-- Load from cloud storage
CREATE TABLE monthly_reports AS
SELECT * FROM 's3://bucket/reports/*.parquet';
```

## Common Financial Analytics Use Cases

### 1. Transaction Analysis
```sql
-- Aggregate transaction volumes by account
SELECT 
    account_id,
    DATE_TRUNC('month', transaction_date) as month,
    COUNT(*) as transaction_count,
    SUM(amount) as total_amount
FROM transactions
GROUP BY account_id, month
ORDER BY month DESC;
```

### 2. Variance Analysis with Window Functions
```sql
-- Period-over-period variance
WITH monthly_summary AS (
    SELECT 
        DATE_TRUNC('month', date) as month,
        SUM(revenue) as total_revenue,
        SUM(expenses) as total_expenses
    FROM financial_data
    GROUP BY month
)
SELECT 
    month,
    total_revenue,
    LAG(total_revenue) OVER (ORDER BY month) as prev_revenue,
    total_revenue - LAG(total_revenue) OVER (ORDER BY month) as variance,
    (total_revenue - LAG(total_revenue) OVER (ORDER BY month)) / 
        LAG(total_revenue) OVER (ORDER BY month) * 100 as variance_pct
FROM monthly_summary;
```

### 3. Reconciliation Data Queries
```sql
-- Find unmatched transactions
SELECT 
    gl.transaction_id,
    gl.amount as gl_amount,
    sub.amount as subledger_amount,
    gl.amount - sub.amount as difference
FROM general_ledger gl
LEFT JOIN subledger sub ON gl.transaction_id = sub.transaction_id
WHERE ABS(gl.amount - sub.amount) > 0.01;
```

### 4. Time-Series Analysis
```sql
-- Rolling averages for financial metrics
SELECT 
    date,
    revenue,
    AVG(revenue) OVER (
        ORDER BY date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as rolling_7day_avg,
    AVG(revenue) OVER (
        ORDER BY date 
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) as rolling_30day_avg
FROM daily_revenue
ORDER BY date;
```

### 5. Export Financial Reports
```sql
-- Export month-end report to Parquet
COPY (
    SELECT 
        account_code,
        account_name,
        SUM(debit) as total_debits,
        SUM(credit) as total_credits,
        SUM(debit) - SUM(credit) as net_balance
    FROM journal_entries
    WHERE DATE_TRUNC('month', entry_date) = '2024-01-01'
    GROUP BY account_code, account_name
) TO 'month_end_report_2024_01.parquet' (FORMAT PARQUET);
```

## Performance Tips

1. **Use Parquet for large datasets**: Columnar format provides better compression and faster queries
2. **Create indexes on frequently queried columns**: Especially for date ranges and account IDs
3. **Partition large tables by date**: Improves query performance for time-range queries
4. **Use COPY for bulk loading**: Much faster than individual INSERT statements
5. **Leverage window functions**: More efficient than self-joins for time-series analysis

## Integration with External Systems

DuckDB can directly query data from:
- **Parquet files** on local disk or cloud storage
- **CSV files** with automatic schema detection
- **JSON/JSONL** for semi-structured data
- **PostgreSQL, MySQL** via database extensions
- **Cloud data warehouses** (Snowflake, BigQuery, Databricks) via federation

This makes DuckDB an excellent choice for:
- ETL/ELT processing
- Data analysis and exploration
- Report generation
- Financial reconciliation
- Audit trail analysis

## Recommended Additional Extensions (Priority P0/P1)

### Data Warehouse Connectors

#### Snowflake Extension
Direct connection to Snowflake for financial data.

```sql
INSTALL snowflake;
LOAD snowflake;

-- Connect to Snowflake data warehouse
ATTACH 'snowflake://account.region/finance_db' AS sf (TYPE snowflake);

-- Query GL data
SELECT account_code, sum(amount) as balance
FROM sf.accounting.general_ledger
WHERE period = '2024-01'
GROUP BY account_code;
```

#### BigQuery Extension
Connect to Google BigQuery for cloud financial data.

```sql
INSTALL bigquery;
LOAD bigquery;

-- Query financial data from BigQuery
ATTACH 'bigquery://finance-project' AS bq (TYPE bigquery);

SELECT transaction_date, sum(amount) as daily_total
FROM bq.finance.transactions
WHERE transaction_date >= '2024-01-01'
GROUP BY transaction_date;
```

#### Databricks Extension
Connect to Databricks for lakehouse-based financial analytics.

```sql
INSTALL databricks;
LOAD databricks;

-- Access financial lakehouse
ATTACH 'databricks://workspace/finance_catalog' AS databricks;

SELECT * FROM databricks.accounting.trial_balance
WHERE fiscal_year = 2024;
```

### Financial Analytics Extensions

#### ANOFOX Statistics Extension
Advanced variance analysis and regression for financial modeling.

```sql
INSTALL anofox_statistics;
LOAD anofox_statistics;

-- Variance decomposition with statistical analysis
WITH monthly_comparison AS (
    SELECT 
        account,
        period,
        actual,
        budget,
        actual - budget as variance,
        LAG(actual, 12) OVER (PARTITION BY account ORDER BY period) as prior_year_actual
    FROM monthly_actuals
)
SELECT 
    account,
    AVG(actual) as avg_actual,
    AVG(budget) as avg_budget,
    AVG(variance) as avg_variance,
    STDDEV(variance) as variance_std_dev,
    -- Year-over-year correlation
    CORR(actual, prior_year_actual) as yoy_correlation
FROM monthly_comparison
WHERE prior_year_actual IS NOT NULL
GROUP BY account;

-- Regression analysis for expense forecasting
SELECT 
    linregr_slope(expense, revenue) as expense_ratio,
    linregr_r2(expense, revenue) as model_fit
FROM historical_financials;
```

#### ANOFOX Forecast Extension
Time series forecasting for financial projections.

```sql
INSTALL anofox_forecast;
LOAD anofox_forecast;

-- Revenue forecasting
SELECT 
    period,
    actual_revenue,
    exponential_smoothing(actual_revenue, 0.3) OVER (ORDER BY period) as smoothed,
    arima_forecast(actual_revenue, 12) OVER (ORDER BY period) as forecast_12mo
FROM monthly_revenue
WHERE period >= '2023-01-01';

-- Cash flow forecasting
SELECT 
    date,
    cash_balance,
    moving_average(cash_balance, 7) OVER (ORDER BY date) as ma_7day,
    holt_winters_forecast(cash_balance, 30) as forecast_30day
FROM daily_cash_position;
```

#### Cache HTTPFS Extension
Cached access to cloud financial data for faster report generation.

```sql
INSTALL cache_httpfs;
LOAD cache_httpfs;

-- Enable caching for frequent report queries
SET cache_httpfs_enabled = true;
SET cache_httpfs_dir = '~/finance-cache';

-- Cached S3 access for month-end reports
SELECT * FROM 's3://finance-data/month-end-reports/2024-*.parquet';
-- Subsequent queries use cache
```

#### Pivot Table Extension
Dynamic financial statement pivots and cross-tabulations.

```sql
INSTALL pivot_table;
LOAD pivot_table;

-- Dynamic P&L by department
PIVOT monthly_expenses
ON department
USING sum(amount) as total_expense
GROUP BY account, month
ORDER BY account, month;

-- Multi-dimensional financial cube
PIVOT (
    SELECT entity, department, account, period, sum(amount) as total
    FROM consolidated_financials
    GROUP BY entity, department, account, period
) ON (department, account)
GROUP BY entity, period;
```

## Extension Priority Matrix

| Priority | Extensions | Purpose |
|----------|-----------|---------|
| **P0 (Critical)** | snowflake, bigquery, databricks | Connect to financial data warehouses |
| **P0 (Critical)** | anofox_statistics | Variance analysis & regression |
| **P1 (High)** | anofox_forecast | Financial forecasting & projections |
| **P1 (High)** | cache_httpfs | Fast cached access to cloud data |
| **P1 (High)** | pivot_table | Dynamic financial pivots |

## Quick Start with Finance Extensions

```sql
-- Install all recommended extensions
INSTALL snowflake;
INSTALL bigquery;
INSTALL databricks;
INSTALL anofox_statistics;
INSTALL anofox_forecast;
INSTALL cache_httpfs;
INSTALL pivot_table;

-- Load for current session
LOAD snowflake;
LOAD bigquery;
LOAD databricks;
LOAD anofox_statistics;
LOAD anofox_forecast;
LOAD cache_httpfs;
LOAD pivot_table;
```

## Resources

- [DuckDB Documentation](https://duckdb.org/docs/)
- [VSS Extension](https://duckdb.org/docs/extensions/vss)
- [Full-Text Search](https://duckdb.org/docs/extensions/full_text_search)
- [Parquet Files](https://duckdb.org/docs/data/parquet/overview)
- [httpfs Extension](https://duckdb.org/docs/extensions/httpfs)

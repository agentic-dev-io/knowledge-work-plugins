# DuckDB Configuration for Data Plugin

## Overview

This plugin uses DuckDB as a high-performance analytical database for data science, BI, and analytics workflows. DuckDB excels at SQL analytics on large datasets with seamless integration to Python/R and cloud data sources.

## Database Location

```
~/knowledge-work-data/data/data.duckdb
```

## Priority Extensions for Data Analytics

This plugin is configured with **Priority P0 (Critical)** extensions for comprehensive data analytics workflows.

## Built-in and Core Extensions

### Parquet (Built-in)
Native support for columnar storage and cloud data lakes.

```sql
-- Read from local Parquet
CREATE TABLE sales_data AS 
SELECT * FROM 'data/sales_*.parquet';

-- Read from S3
CREATE TABLE cloud_data AS
SELECT * FROM 's3://my-bucket/data/*.parquet';

-- Write query results to Parquet
COPY (
    SELECT date, product, SUM(revenue) as total_revenue
    FROM sales_data
    GROUP BY date, product
) TO 'output/aggregated.parquet' (FORMAT PARQUET);
```

### VSS (Vector Similarity Search)
Embedding-based analytics and ML feature search.

```sql
INSTALL vss;
LOAD vss;

-- Store feature embeddings
CREATE TABLE feature_embeddings (
    feature_id INT,
    feature_name VARCHAR,
    embedding FLOAT[128]
);

CREATE INDEX features_idx ON feature_embeddings 
USING HNSW (embedding) WITH (metric = 'cosine');

-- Find similar features
SELECT feature_id, feature_name,
       array_cosine_similarity(embedding, $target_embedding::FLOAT[128]) as similarity
FROM feature_embeddings
ORDER BY similarity DESC LIMIT 20;
```

### JSON Extension
Handle semi-structured data from APIs and logs.

```sql
INSTALL json;
LOAD json;

-- Parse nested JSON from logs
SELECT 
    json_extract(log_data, '$.timestamp') as timestamp,
    json_extract(log_data, '$.user.id') as user_id,
    json_extract(log_data, '$.event.type') as event_type
FROM application_logs;
```

### HTTPFS Extension
Direct access to cloud storage without downloads.

```sql
INSTALL httpfs;
LOAD httpfs;

-- Configure S3
SET s3_region='us-east-1';
SET s3_access_key_id='your-key';
SET s3_secret_access_key='your-secret';

-- Query directly from S3
SELECT * FROM 's3://analytics-bucket/events/year=2024/month=01/*.parquet'
WHERE event_type = 'purchase';
```

### Statistics Extensions
Advanced statistical functions.

```sql
-- Correlation analysis
SELECT 
    corr(feature1, target) as feature1_corr,
    corr(feature2, target) as feature2_corr,
    corr(feature3, target) as feature3_corr
FROM ml_dataset;

-- Percentiles
SELECT 
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY value) as q25,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY value) as median,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY value) as q75,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY value) as q95
FROM metrics;
```

## Common Data Analytics Use Cases

### 1. Time Series Analysis
```sql
-- Moving averages and trends
SELECT 
    date,
    metric_value,
    AVG(metric_value) OVER (
        ORDER BY date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as rolling_7day,
    metric_value - LAG(metric_value, 7) OVER (ORDER BY date) as week_over_week_change,
    (metric_value - LAG(metric_value, 7) OVER (ORDER BY date)) * 100.0 / 
        LAG(metric_value, 7) OVER (ORDER BY date) as wow_change_pct
FROM daily_metrics
ORDER BY date;
```

### 2. Cohort Analysis
```sql
-- User retention cohorts
WITH cohorts AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', first_seen) as cohort_month,
        DATE_TRUNC('month', activity_date) as activity_month
    FROM user_activity
)
SELECT 
    cohort_month,
    EXTRACT(MONTH FROM AGE(activity_month, cohort_month)) as months_since_signup,
    COUNT(DISTINCT user_id) as active_users
FROM cohorts
GROUP BY cohort_month, months_since_signup
ORDER BY cohort_month, months_since_signup;
```

### 3. Funnel Analysis
```sql
-- Conversion funnel
WITH funnel_events AS (
    SELECT 
        user_id,
        MAX(CASE WHEN event = 'page_view' THEN 1 ELSE 0 END) as viewed,
        MAX(CASE WHEN event = 'add_to_cart' THEN 1 ELSE 0 END) as added_cart,
        MAX(CASE WHEN event = 'checkout' THEN 1 ELSE 0 END) as checkout,
        MAX(CASE WHEN event = 'purchase' THEN 1 ELSE 0 END) as purchased
    FROM events
    WHERE date >= CURRENT_DATE - INTERVAL 7 DAY
    GROUP BY user_id
)
SELECT 
    SUM(viewed) as step1_views,
    SUM(added_cart) as step2_add_cart,
    SUM(checkout) as step3_checkout,
    SUM(purchased) as step4_purchase,
    SUM(added_cart) * 100.0 / SUM(viewed) as view_to_cart_rate,
    SUM(checkout) * 100.0 / SUM(added_cart) as cart_to_checkout_rate,
    SUM(purchased) * 100.0 / SUM(checkout) as checkout_to_purchase_rate
FROM funnel_events;
```

### 4. A/B Test Analysis
```sql
-- Statistical significance testing
WITH test_metrics AS (
    SELECT 
        variant,
        COUNT(DISTINCT user_id) as users,
        SUM(converted) as conversions,
        AVG(revenue) as avg_revenue,
        STDDEV(revenue) as stddev_revenue
    FROM ab_test_results
    GROUP BY variant
)
SELECT 
    variant,
    users,
    conversions,
    conversions * 100.0 / users as conversion_rate,
    avg_revenue,
    -- Z-score for conversion rate
    (MAX(CASE WHEN variant = 'treatment' THEN conversions * 1.0 / users END) OVER () - 
     MAX(CASE WHEN variant = 'control' THEN conversions * 1.0 / users END) OVER ()) /
    SQRT(
        MAX(CASE WHEN variant = 'treatment' THEN conversions * 1.0 / users * (1 - conversions * 1.0 / users) / users END) OVER () +
        MAX(CASE WHEN variant = 'control' THEN conversions * 1.0 / users * (1 - conversions * 1.0 / users) / users END) OVER ()
    ) as z_score
FROM test_metrics;
```

### 5. Feature Engineering
```sql
-- Create ML features
CREATE TABLE ml_features AS
SELECT 
    user_id,
    -- Recency features
    EXTRACT(DAY FROM (CURRENT_DATE - MAX(event_date))) as days_since_last_activity,
    -- Frequency features
    COUNT(*) as total_events,
    COUNT(DISTINCT DATE_TRUNC('day', event_date)) as active_days,
    -- Monetary features
    SUM(CASE WHEN event_type = 'purchase' THEN amount ELSE 0 END) as total_revenue,
    AVG(CASE WHEN event_type = 'purchase' THEN amount END) as avg_purchase,
    -- Engagement features
    COUNT(DISTINCT event_type) as event_diversity,
    MAX(session_duration) as longest_session
FROM user_events
GROUP BY user_id;
```

## Performance Optimization

### 1. Partitioning
```sql
-- Partition by date for time-series queries
CREATE TABLE events_partitioned AS
SELECT * FROM read_parquet('data/events/*.parquet', 
    hive_partitioning = true);
```

### 2. Indexes
```sql
-- Create indexes on frequently filtered columns
CREATE INDEX idx_user_id ON events(user_id);
CREATE INDEX idx_date ON events(event_date);
```

### 3. Query Optimization
```sql
-- Use EXPLAIN to analyze query plans
EXPLAIN SELECT * FROM large_table WHERE condition;

-- Use CTEs for complex queries
WITH filtered AS (
    SELECT * FROM events WHERE date >= '2024-01-01'
)
SELECT * FROM filtered WHERE user_id IN (SELECT user_id FROM active_users);
```

## Integration with Data Tools

DuckDB integrates seamlessly with:
- **Python**: `duckdb` package, pandas, polars, arrow
- **R**: `duckdb` package, dbplyr, arrow
- **BI Tools**: Tableau, Power BI, Metabase via ODBC/JDBC
- **Notebooks**: Jupyter, VS Code, DataSpell
- **Cloud**: S3, GCS, Azure Blob Storage via httpfs

## Recommended Additional Extensions (Priority P0)

### Data Warehouse Connectors

#### Snowflake Extension
Direct queries to Snowflake data warehouse.

```sql
INSTALL snowflake;
LOAD snowflake;

-- Query Snowflake directly
ATTACH 'snowflake://account.region/database' AS sf (TYPE snowflake);
SELECT * FROM sf.schema.table LIMIT 100;
```

#### BigQuery Extension
Connect to Google BigQuery for cloud data warehouse queries.

```sql
INSTALL bigquery;
LOAD bigquery;

-- Query BigQuery
ATTACH 'bigquery://project-id' AS bq (TYPE bigquery);
SELECT * FROM bq.dataset.table WHERE date >= '2024-01-01';
```

#### Databricks Extension
Connect to Databricks lakehouse platform.

```sql
INSTALL databricks;
LOAD databricks;

-- Query Databricks
ATTACH 'databricks://workspace-url/catalog' AS databricks;
SELECT * FROM databricks.schema.table;
```

### Advanced Analytics Extensions

#### ANOFOX Statistics Extension
Statistical analysis including regression and correlation.

```sql
INSTALL anofox_statistics;
LOAD anofox_statistics;

-- Linear regression
SELECT linregr_slope(y, x) as slope,
       linregr_intercept(y, x) as intercept,
       linregr_r2(y, x) as r_squared
FROM dataset;

-- Multiple correlation analysis
SELECT 
    CORR(col1, col2) as corr_col1_col2,
    CORR(col1, col3) as corr_col1_col3,
    CORR(col1, col4) as corr_col1_col4,
    CORR(col2, col3) as corr_col2_col3,
    CORR(col2, col4) as corr_col2_col4,
    CORR(col3, col4) as corr_col3_col4
FROM dataset;
```

#### ANOFOX Forecast Extension
Time series forecasting and trend analysis.

```sql
INSTALL anofox_forecast;
LOAD anofox_forecast;

-- Exponential smoothing forecast
SELECT date,
       actual_value,
       exponential_smoothing(actual_value, 0.3) OVER (ORDER BY date) as forecast
FROM time_series;

-- ARIMA forecasting
SELECT arima_forecast(value, 30) as forecast_30_days
FROM daily_metrics
WHERE date >= CURRENT_DATE - INTERVAL 90 DAY;
```

### Vector Search Extensions

#### FAISS Extension
High-performance vector similarity search for large-scale embeddings.

```sql
INSTALL faiss;
LOAD faiss;

-- Create FAISS index for massive embeddings
CREATE FAISS INDEX ON embeddings_table(embedding_column)
WITH (index_type = 'IVF', nlist = 100);

-- Fast similarity search
SELECT id, embedding_cosine_distance(embedding, $query_vector) as distance
FROM embeddings_table
WHERE faiss_search(embedding, $query_vector, 100);
```

#### Quackformers Extension
Transformer models and NLP directly in DuckDB.

```sql
INSTALL quackformers;
LOAD quackformers;

-- Generate embeddings from text
SELECT id, text,
       sentence_embedding(text, 'all-MiniLM-L6-v2') as embedding
FROM documents;

-- Sentiment analysis
SELECT review_text,
       sentiment_analysis(review_text) as sentiment_score
FROM customer_reviews;
```

### Visualization & Analysis

#### Dash Extension
Interactive dashboard components.

```sql
INSTALL dash;
LOAD dash;

-- Create dashboard data
SELECT date,
       sum(revenue) as revenue,
       count(distinct user_id) as users,
       dash_sparkline(revenue) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as trend
FROM events
GROUP BY date;
```

#### Miniplot Extension
ASCII/Unicode plots in query results.

```sql
INSTALL miniplot;
LOAD miniplot;

-- Generate histogram in results
SELECT value_bucket,
       count(*) as frequency,
       bar(count(*), 50) as distribution
FROM (
    SELECT width_bucket(value, 0, 100, 10) as value_bucket
    FROM measurements
)
GROUP BY value_bucket
ORDER BY value_bucket;
```

#### Pivot Table Extension
Dynamic pivot operations.

```sql
INSTALL pivot_table;
LOAD pivot_table;

-- Dynamic pivot
PIVOT sales_data 
ON product 
USING sum(revenue) as total_revenue
GROUP BY region, quarter;

-- Multi-level pivot
PIVOT (
    SELECT region, category, product, sum(sales) as total
    FROM sales
    GROUP BY region, category, product
) ON (category, product)
GROUP BY region;
```

### SQL Analysis

#### Parser Tools Extension
SQL query parsing, analysis, and optimization.

```sql
INSTALL parser_tools;
LOAD parser_tools;

-- Parse SQL query
SELECT parse_sql('SELECT * FROM table WHERE x > 10') as parsed_ast;

-- Analyze query complexity
SELECT query_complexity_score($your_query) as complexity,
       query_tables($your_query) as tables_used,
       query_columns($your_query) as columns_accessed;

-- Suggest query optimizations
SELECT optimize_query($your_query) as optimized_sql;
```

## Extension Priority Matrix

| Priority | Extensions | Purpose |
|----------|-----------|---------|
| **P0 (Critical)** | snowflake, bigquery, databricks | Data warehouse connectivity |
| **P0 (Critical)** | anofox_statistics, anofox_forecast | Advanced analytics & forecasting |
| **P0 (Critical)** | faiss, quackformers | Vector search & NLP |
| **P1 (High)** | dash, miniplot, pivot_table | Visualization & exploration |
| **P1 (High)** | parser_tools | SQL analysis & optimization |

## Quick Start with Extensions

```sql
-- Install all priority extensions
INSTALL snowflake;
INSTALL bigquery;
INSTALL databricks;
INSTALL anofox_statistics;
INSTALL anofox_forecast;
INSTALL faiss;
INSTALL quackformers;
INSTALL dash;
INSTALL miniplot;
INSTALL pivot_table;
INSTALL parser_tools;

-- Load extensions for current session
LOAD snowflake;
LOAD bigquery;
LOAD anofox_statistics;
LOAD anofox_forecast;
LOAD faiss;
```

## Resources

- [DuckDB Documentation](https://duckdb.org/docs/)
- [Python API](https://duckdb.org/docs/api/python/overview)
- [SQL Reference](https://duckdb.org/docs/sql/introduction)
- [Extensions](https://duckdb.org/docs/extensions/overview)

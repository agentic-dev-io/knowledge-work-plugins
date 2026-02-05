# DuckDB Configuration for Product Management Plugin

## Overview

This plugin uses DuckDB for product metrics tracking, KPI analysis, and product analytics. DuckDB enables fast queries on feature usage data, user feedback, and roadmap metrics.

## Database Location

```
~/knowledge-work-data/product-management/product-management.duckdb
```

## Key Extensions

### VSS (Vector Similarity Search)
Find similar feature requests, user feedback, and product requirements.

```sql
INSTALL vss;
LOAD vss;

-- Store feature request embeddings
CREATE TABLE feature_requests (
    request_id INT,
    title VARCHAR,
    description TEXT,
    embedding FLOAT[384]
);

CREATE INDEX requests_idx ON feature_requests 
USING HNSW (embedding) WITH (metric = 'cosine');

-- Find similar requests
SELECT request_id, title,
       array_cosine_similarity(embedding, $1::FLOAT[384]) as similarity
FROM feature_requests
ORDER BY similarity DESC LIMIT 10;
```

### Full-Text Search
Search through user feedback, feature descriptions, and product documentation.

```sql
INSTALL fts;
LOAD fts;

PRAGMA create_fts_index('feature_requests', 'request_id', 'title', 'description');

SELECT * FROM (
    SELECT *, fts_main_feature_requests.match_bm25(request_id, 'mobile authentication') AS score
    FROM feature_requests
) WHERE score IS NOT NULL ORDER BY score DESC;
```

### Parquet for Analytics Data
Efficiently store and query product usage metrics and event data.

```sql
-- Load usage events
CREATE TABLE usage_events AS 
SELECT * FROM 'analytics/events_*.parquet';

-- Aggregate by feature
SELECT 
    feature_name,
    DATE_TRUNC('week', event_timestamp) as week,
    COUNT(DISTINCT user_id) as active_users,
    COUNT(*) as event_count
FROM usage_events
GROUP BY feature_name, week
ORDER BY week DESC;
```

## Common Product Management Queries

### Feature Adoption Metrics
```sql
-- Track feature adoption over time
WITH feature_users AS (
    SELECT 
        feature_id,
        DATE_TRUNC('week', first_used_at) as adoption_week,
        COUNT(DISTINCT user_id) as new_users
    FROM feature_usage
    GROUP BY feature_id, adoption_week
)
SELECT 
    feature_id,
    adoption_week,
    new_users,
    SUM(new_users) OVER (PARTITION BY feature_id ORDER BY adoption_week) as cumulative_users
FROM feature_users
ORDER BY feature_id, adoption_week;
```

### User Retention Analysis
```sql
-- Cohort retention analysis
SELECT 
    DATE_TRUNC('month', signup_date) as cohort_month,
    COUNT(DISTINCT user_id) as cohort_size,
    COUNT(DISTINCT CASE WHEN last_active_date >= signup_date + INTERVAL 30 DAY THEN user_id END) as retained_30d,
    COUNT(DISTINCT CASE WHEN last_active_date >= signup_date + INTERVAL 30 DAY THEN user_id END) * 100.0 / COUNT(DISTINCT user_id) as retention_rate
FROM users
GROUP BY cohort_month
ORDER BY cohort_month DESC;
```

### Roadmap Progress Tracking
```sql
-- Track feature delivery vs. plan
SELECT 
    sprint_name,
    COUNT(*) as total_features,
    SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) as completed,
    SUM(CASE WHEN status = 'completed' THEN story_points ELSE 0 END) as completed_points,
    SUM(story_points) as total_points,
    SUM(CASE WHEN status = 'completed' THEN story_points ELSE 0 END) * 100.0 / SUM(story_points) as completion_rate
FROM roadmap_items
GROUP BY sprint_name
ORDER BY sprint_name;
```

## Recommended Additional Extensions (Priority P1/P2)

### Web & Content Processing

#### Webbed Extension
XML/HTML processing for competitor research and web scraping.

```sql
INSTALL webbed;
LOAD webbed;

-- Parse competitor product pages
SELECT 
    url,
    xpath(html_content, '//h1[@class="product-title"]') as product_name,
    xpath(html_content, '//div[@class="features"]/ul/li') as features
FROM competitor_pages;

-- Extract pricing from web data
SELECT 
    competitor,
    cast(xpath(html, '//span[@class="price"]')[1] as DECIMAL) as price
FROM scraped_pricing;
```

#### Markdown Extension
Process and generate product specs in Markdown format.

```sql
INSTALL markdown;
LOAD markdown;

-- Generate spec documentation
SELECT 
    feature_id,
    feature_name,
    markdown_to_html(spec_content) as html_spec,
    markdown_toc(spec_content) as table_of_contents
FROM feature_specs;

-- Parse markdown PRDs
SELECT 
    prd_id,
    extract_headers(markdown_content, 2) as h2_sections,
    extract_code_blocks(markdown_content) as code_examples
FROM product_requirements;
```

### Data Transformation

#### YAML Extension
Parse roadmap configurations and workflow definitions.

```sql
INSTALL yaml;
LOAD yaml;

-- Parse roadmap YAML config
SELECT 
    yaml_extract(roadmap_config, '$.quarters[*].themes') as quarterly_themes,
    yaml_extract(roadmap_config, '$.priorities') as priorities
FROM roadmap_definitions;

-- Load feature flag configurations
SELECT 
    feature_name,
    yaml_extract(config, '$.enabled') as is_enabled,
    yaml_extract(config, '$.rollout_percentage') as rollout_pct
FROM feature_flags;
```

#### JSONata Extension
Transform JSON data from research tools and analytics platforms.

```sql
INSTALL jsonata;
LOAD jsonata;

-- Transform user research data
SELECT 
    jsonata_transform(interview_data, 
        '{ "user": user_id, "insights": feedback[sentiment="positive"].text }') as processed
FROM user_interviews;

-- Aggregate feature requests from multiple tools
SELECT 
    jsonata_transform(api_response,
        'requests[].{ "id": id, "votes": votes, "priority": $priority(votes) }') as normalized
FROM external_feedback;
```

### Analytics Extensions

#### DataSketches Extension
Approximate distinct counts for user metrics and feature usage.

```sql
INSTALL datasketches;
LOAD datasketches;

-- Approximate unique users per feature (memory efficient)
SELECT 
    feature_id,
    theta_sketch_distinct(user_id) as approx_unique_users,
    theta_sketch_union(user_id) OVER (PARTITION BY product_area) as area_total_users
FROM feature_usage
GROUP BY feature_id;

-- HyperLogLog for large-scale cardinality
SELECT 
    date,
    hll_sketch_count(distinct user_id) as daily_active_users
FROM events
GROUP BY date;
```

#### Pivot Table Extension
Roadmap prioritization and feature analysis matrices.

```sql
INSTALL pivot_table;
LOAD pivot_table;

-- Feature priority matrix
PIVOT feature_requests
ON (impact, effort)
USING count(*) as request_count
GROUP BY product_area;

-- Roadmap timeline pivot
PIVOT (
    SELECT quarter, team, feature, status
    FROM roadmap
) ON team
USING count(feature) as feature_count
GROUP BY quarter, status;
```

## Extension Priority Matrix

| Priority | Extensions | Purpose |
|----------|-----------|---------|
| **P1 (High)** | webbed | Competitor research & web scraping |
| **P1 (High)** | markdown | Spec documentation |
| **P1 (High)** | jsonata | Transform research tool data |
| **P2 (Medium)** | yaml | Roadmap & workflow configuration |
| **P2 (Medium)** | datasketches | Approximate user metrics |
| **P2 (Medium)** | pivot_table | Roadmap analysis matrices |

## Quick Start with Product Management Extensions

```sql
-- Install all recommended extensions
INSTALL webbed;
INSTALL markdown;
INSTALL yaml;
INSTALL jsonata;
INSTALL datasketches;
INSTALL pivot_table;

-- Load for current session
LOAD webbed;
LOAD markdown;
LOAD yaml;
LOAD jsonata;
LOAD datasketches;
LOAD pivot_table;
```

## Resources

- [DuckDB Documentation](https://duckdb.org/docs/)
- [Time Series Analysis](https://duckdb.org/docs/sql/functions/date)
- [Window Functions](https://duckdb.org/docs/sql/window_functions)

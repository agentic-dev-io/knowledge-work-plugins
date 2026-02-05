# DuckDB Configuration for Sales Plugin

## Overview

This plugin uses DuckDB for sales pipeline analytics, deal tracking, customer relationship analysis, and sales forecasting with graph-like relationship queries.

## Database Location

```
~/knowledge-work-data/sales/sales.duckdb
```

## Key Extensions

### VSS (Vector Similarity Search)
Find similar deals, prospects, and sales opportunities.

```sql
INSTALL vss;
LOAD vss;

CREATE TABLE opportunities (
    opp_id INT,
    account_name VARCHAR,
    deal_size DECIMAL,
    stage VARCHAR,
    close_date DATE,
    notes TEXT,
    embedding FLOAT[384]
);

CREATE INDEX opps_idx ON opportunities 
USING HNSW (embedding) WITH (metric = 'cosine');

-- Find similar deals for insights
SELECT opp_id, account_name, deal_size, stage,
       array_cosine_similarity(embedding, $current_deal::FLOAT[384]) as similarity
FROM opportunities
WHERE stage = 'closed_won'
ORDER BY similarity DESC LIMIT 10;
```

### JSON for Hierarchical Data
Store account hierarchies and relationships.

```sql
INSTALL json;
LOAD json;

CREATE TABLE accounts (
    account_id INT,
    account_name VARCHAR,
    hierarchy JSON  -- Store parent-child relationships
);

-- Query account hierarchies
SELECT 
    account_id,
    account_name,
    json_extract(hierarchy, '$.parent_id') as parent_account,
    json_extract(hierarchy, '$.level') as org_level
FROM accounts;
```

## Common Sales Queries

### Pipeline Analysis
```sql
-- Pipeline by stage and rep
SELECT 
    sales_rep,
    stage,
    COUNT(*) as deal_count,
    SUM(deal_value) as total_value,
    AVG(deal_value) as avg_deal_size,
    AVG(EXTRACT(DAY FROM (CURRENT_DATE - created_date))) as avg_age_days
FROM opportunities
WHERE stage != 'closed_lost' AND stage != 'closed_won'
GROUP BY sales_rep, stage
ORDER BY sales_rep, total_value DESC;
```

### Win Rate Analysis
```sql
-- Win rates by segment and size
SELECT 
    customer_segment,
    CASE 
        WHEN deal_value < 10000 THEN 'Small'
        WHEN deal_value < 50000 THEN 'Medium'
        ELSE 'Large'
    END as deal_size_category,
    COUNT(*) as total_opportunities,
    SUM(CASE WHEN stage = 'closed_won' THEN 1 ELSE 0 END) as wins,
    SUM(CASE WHEN stage = 'closed_won' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as win_rate,
    AVG(CASE WHEN stage = 'closed_won' THEN deal_value END) as avg_win_value
FROM opportunities
WHERE stage IN ('closed_won', 'closed_lost')
GROUP BY customer_segment, deal_size_category;
```

### Sales Velocity
```sql
-- Average time in each stage
SELECT 
    stage,
    COUNT(*) as deals_in_stage,
    AVG(EXTRACT(DAY FROM (stage_exit_date - stage_entry_date))) as avg_days_in_stage,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY EXTRACT(DAY FROM (stage_exit_date - stage_entry_date))) as median_days
FROM stage_history
WHERE stage_exit_date IS NOT NULL
GROUP BY stage
ORDER BY 
    CASE stage
        WHEN 'prospecting' THEN 1
        WHEN 'qualification' THEN 2
        WHEN 'proposal' THEN 3
        WHEN 'negotiation' THEN 4
        WHEN 'closed_won' THEN 5
    END;
```

### Forecast Accuracy
```sql
-- Compare forecasts to actuals
WITH forecast_vs_actual AS (
    SELECT 
        forecast_month,
        SUM(forecasted_value) as forecasted,
        SUM(CASE WHEN closed_date IS NOT NULL THEN actual_value ELSE 0 END) as actual
    FROM sales_forecast
    GROUP BY forecast_month
)
SELECT 
    forecast_month,
    forecasted,
    actual,
    actual - forecasted as variance,
    (actual - forecasted) * 100.0 / forecasted as variance_pct
FROM forecast_vs_actual
ORDER BY forecast_month DESC;
```

## Recommended Additional Extensions (Priority P1/P2)

### Web Research & Prospecting

#### Webbed Extension
Web scraping for prospect and company research.

```sql
INSTALL webbed;
LOAD webbed;

-- Extract company information from websites
SELECT 
    company_url,
    xpath(html_content, '//meta[@property="og:description"]/@content') as company_description,
    xpath(html_content, '//div[@class="team-size"]') as employee_count
FROM prospect_websites;

-- Parse LinkedIn company pages
SELECT 
    linkedin_url,
    extract_json_ld(html) as structured_data,
    xpath(html, '//span[@class="company-industry"]') as industry
FROM linkedin_research;
```

#### Netquack Extension
URI/Domain/IP parsing for lead enrichment.

```sql
INSTALL netquack;
LOAD netquack;

-- Parse and enrich email domains
SELECT 
    email,
    extract_domain(email) as company_domain,
    domain_to_company_name(extract_domain(email)) as likely_company,
    is_corporate_email(email) as is_business_email
FROM leads;

-- Analyze prospect website URLs
SELECT 
    url,
    parse_url(url).host as domain,
    parse_url(url).path as page_path,
    classify_domain(parse_url(url).host) as domain_type
FROM prospect_interactions;
```

### Data Processing

#### JSONata Extension
Transform CRM API data and enrich lead information.

```sql
INSTALL jsonata;
LOAD jsonata;

-- Transform CRM contact data
SELECT 
    contact_id,
    jsonata_transform(crm_data,
        '{ "name": firstName & " " & lastName, 
           "company": account.name,
           "score": leadScore + accountScore }') as enriched_contact
FROM crm_contacts;

-- Parse ZoomInfo API responses
SELECT 
    jsonata_transform(api_response,
        'contacts[].{ 
            "name": name, 
            "title": title, 
            "phone": phones[type="work"][0].number,
            "seniority": $seniority_level(title)
        }') as processed_contacts
FROM zoominfo_results;
```

### Fuzzy Matching & Deduplication

#### Fuzzycomplete Extension
Fuzzy matching for lead deduplication and account matching.

```sql
INSTALL fuzzycomplete;
LOAD fuzzycomplete;

-- Deduplicate leads with fuzzy matching
SELECT 
    l1.lead_id,
    l1.company_name,
    l2.lead_id as potential_duplicate,
    fuzzy_match_score(l1.company_name, l2.company_name) as similarity
FROM leads l1, leads l2
WHERE l1.lead_id < l2.lead_id
  AND fuzzy_match_score(l1.company_name, l2.company_name) > 0.85;

-- Auto-complete for account search
SELECT 
    account_name,
    fuzzy_autocomplete(account_name, $search_term) as match_score
FROM accounts
WHERE fuzzy_autocomplete(account_name, $search_term) > 0.7
ORDER BY match_score DESC
LIMIT 10;
```

#### Marisa Extension
Fast trie-based lookups for contact and account history.

```sql
INSTALL marisa;
LOAD marisa;

-- Fast prefix search for contacts
CREATE MARISA TRIE contact_trie ON contacts(full_name);

SELECT * FROM contacts
WHERE marisa_prefix_search(full_name, 'John');

-- Quick account lookup by name prefix
CREATE MARISA TRIE account_trie ON accounts(account_name);

SELECT account_id, account_name, annual_revenue
FROM accounts
WHERE marisa_prefix_search(account_name, $partial_name)
ORDER BY annual_revenue DESC;
```

## Extension Priority Matrix

| Priority | Extensions | Purpose |
|----------|-----------|---------|
| **P1 (High)** | webbed | Company & prospect research |
| **P1 (High)** | jsonata | CRM data transformation |
| **P1 (High)** | fuzzycomplete | Lead deduplication |
| **P2 (Medium)** | netquack | Email/domain parsing |
| **P2 (Medium)** | marisa | Fast contact/account lookups |

## Quick Start with Sales Extensions

```sql
-- Install all recommended extensions
INSTALL webbed;
INSTALL netquack;
INSTALL jsonata;
INSTALL fuzzycomplete;
INSTALL marisa;

-- Load for current session
LOAD webbed;
LOAD netquack;
LOAD jsonata;
LOAD fuzzycomplete;
LOAD marisa;
```

## Resources

- [DuckDB Window Functions](https://duckdb.org/docs/sql/window_functions)
- [Date/Time Functions](https://duckdb.org/docs/sql/functions/date)
- [VSS Extension](https://duckdb.org/docs/extensions/vss)

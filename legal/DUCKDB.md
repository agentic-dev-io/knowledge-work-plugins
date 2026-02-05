# DuckDB Configuration for Legal Plugin

## Overview

This plugin uses DuckDB for contract analysis, document management, and legal research with full-text search and document similarity capabilities.

## Database Location

```
~/knowledge-work-data/legal/legal.duckdb
```

## Key Extensions

### VSS (Vector Similarity Search)
Find similar contracts, clauses, and legal precedents.

```sql
INSTALL vss;
LOAD vss;

CREATE TABLE contracts (
    contract_id INT,
    contract_name VARCHAR,
    contract_type VARCHAR,
    party_names TEXT,
    terms TEXT,
    embedding FLOAT[768]
);

CREATE INDEX contracts_idx ON contracts 
USING HNSW (embedding) WITH (metric = 'cosine');

-- Find similar contracts for reference
SELECT contract_id, contract_name, contract_type,
       array_cosine_similarity(embedding, $current_contract::FLOAT[768]) as similarity
FROM contracts
WHERE contract_type = 'NDA'
ORDER BY similarity DESC LIMIT 10;
```

### Full-Text Search
Search contract terms, clauses, and legal documents.

```sql
INSTALL fts;
LOAD fts;

PRAGMA create_fts_index('contracts', 'contract_id', 'terms', 'party_names');

-- Search for specific clauses
SELECT contract_id, contract_name, score
FROM (
    SELECT *, fts_main_contracts.match_bm25(contract_id, 'indemnification liability') AS score
    FROM contracts
) WHERE score IS NOT NULL ORDER BY score DESC;
```

### JSON for Structured Legal Data
Store contract metadata and clause hierarchies.

```sql
INSTALL json;
LOAD json;

CREATE TABLE contract_metadata (
    contract_id INT,
    metadata JSON
);

-- Query contract terms
SELECT 
    contract_id,
    json_extract(metadata, '$.effective_date') as effective_date,
    json_extract(metadata, '$.expiration_date') as expiration_date,
    json_extract(metadata, '$.jurisdiction') as jurisdiction,
    json_extract_string(metadata, '$.clauses[*].type') as clause_types
FROM contract_metadata;
```

## Common Legal Queries

### Contract Expiration Tracking
```sql
-- Upcoming contract renewals
SELECT 
    contract_id,
    contract_name,
    party_names,
    expiration_date,
    EXTRACT(DAY FROM (expiration_date - CURRENT_DATE)) as days_until_expiration,
    contract_value,
    auto_renew
FROM contracts
WHERE expiration_date BETWEEN CURRENT_DATE AND CURRENT_DATE + INTERVAL 90 DAY
ORDER BY days_until_expiration ASC;
```

### Risk Analysis
```sql
-- Identify high-risk contract provisions
WITH risk_clauses AS (
    SELECT contract_id, COUNT(*) as risk_count
    FROM contract_clauses
    WHERE clause_type IN ('unlimited_liability', 'no_cap_damages', 'broad_indemnification')
    GROUP BY contract_id
)
SELECT 
    c.contract_id,
    c.contract_name,
    c.counterparty,
    c.contract_value,
    r.risk_count,
    CASE 
        WHEN r.risk_count >= 3 THEN 'High Risk'
        WHEN r.risk_count >= 1 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END as risk_level
FROM contracts c
LEFT JOIN risk_clauses r ON c.contract_id = r.contract_id
ORDER BY r.risk_count DESC NULLS LAST;
```

### Document Review Pipeline
```sql
-- Track review status
SELECT 
    document_type,
    status,
    COUNT(*) as document_count,
    AVG(EXTRACT(DAY FROM (CURRENT_TIMESTAMP - submitted_at))) as avg_age_days,
    SUM(CASE WHEN priority = 'urgent' THEN 1 ELSE 0 END) as urgent_count
FROM legal_documents
WHERE status NOT IN ('approved', 'rejected')
GROUP BY document_type, status
ORDER BY urgent_count DESC, avg_age_days DESC;
```

### Clause Library Management
```sql
-- Standard clause usage across contracts
SELECT 
    clause_type,
    COUNT(DISTINCT contract_id) as used_in_contracts,
    AVG(LENGTH(clause_text)) as avg_clause_length,
    MAX(last_updated) as most_recent_update
FROM standard_clauses
GROUP BY clause_type
ORDER BY used_in_contracts DESC;
```

## Recommended Additional Extensions (Priority P2/P3)

### Document Processing

#### Markdown Extension
Process and generate legal documentation and contract templates.

```sql
INSTALL markdown;
LOAD markdown;

-- Generate contract templates in Markdown
SELECT 
    template_id,
    template_name,
    markdown_to_html(template_content) as html_version,
    extract_markdown_variables(template_content) as required_fields
FROM contract_templates;

-- Parse legal documentation
SELECT 
    doc_id,
    extract_markdown_headers(content, 2) as sections,
    markdown_toc(content) as table_of_contents,
    markdown_word_count(content) as word_count
FROM legal_memos;
```

#### YAML Extension
Manage compliance checklists and legal workflow configurations.

```sql
INSTALL yaml;
LOAD yaml;

-- Parse compliance checklists
SELECT 
    regulation,
    yaml_extract(checklist, '$.requirements[*].description') as requirements,
    yaml_extract(checklist, '$.risk_level') as risk_level
FROM compliance_configs;

-- Load contract review workflows
SELECT 
    contract_type,
    yaml_extract(workflow, '$.review_stages') as review_stages,
    yaml_extract(workflow, '$.approval_matrix') as approvers
FROM review_workflows;
```

### Fuzzy Matching

#### Rapidfuzz Extension
Find similar NDA clauses and contract provisions.

```sql
INSTALL rapidfuzz;
LOAD rapidfuzz;

-- Match NDA clauses across contracts
SELECT 
    c1.contract_id as contract_1,
    c1.clause_text as clause_1,
    c2.contract_id as contract_2,
    c2.clause_text as clause_2,
    rapidfuzz_similarity(c1.clause_text, c2.clause_text) as similarity
FROM contract_clauses c1, contract_clauses c2
WHERE c1.clause_type = 'confidentiality'
  AND c2.clause_type = 'confidentiality'
  AND c1.contract_id < c2.contract_id
  AND rapidfuzz_similarity(c1.clause_text, c2.clause_text) > 0.85;

-- Find similar legal terms
SELECT 
    term,
    definition,
    rapidfuzz_partial_ratio(definition, $search_definition) as match_score
FROM legal_glossary
WHERE rapidfuzz_partial_ratio(definition, $search_definition) > 75
ORDER BY match_score DESC;
```

### Text Encoding

#### Encoding Extension
Handle documents from various sources with different text encodings.

```sql
INSTALL encoding;
LOAD encoding;

-- Convert document encodings
SELECT 
    document_id,
    convert_encoding(content, 'ISO-8859-1', 'UTF-8') as utf8_content,
    detect_encoding(content) as detected_encoding
FROM legacy_documents;

-- Normalize text from different sources
SELECT 
    source,
    document_name,
    CASE detect_encoding(raw_content)
        WHEN 'UTF-16' THEN convert_encoding(raw_content, 'UTF-16', 'UTF-8')
        WHEN 'ISO-8859-1' THEN convert_encoding(raw_content, 'ISO-8859-1', 'UTF-8')
        ELSE raw_content
    END as normalized_content
FROM scanned_contracts;
```

## Extension Priority Matrix

| Priority | Extensions | Purpose |
|----------|-----------|---------|
| **P2 (Medium)** | markdown | Contract templates & documentation |
| **P2 (Medium)** | yaml | Compliance checklists & workflows |
| **P2 (Medium)** | rapidfuzz | NDA & clause matching |
| **P3 (Nice-to-have)** | encoding | Handle legacy document encodings |

## Quick Start with Legal Extensions

```sql
-- Install all recommended extensions
INSTALL markdown;
INSTALL yaml;
INSTALL rapidfuzz;
INSTALL encoding;

-- Load for current session
LOAD markdown;
LOAD yaml;
LOAD rapidfuzz;
LOAD encoding;
```

## Resources

- [DuckDB Full-Text Search](https://duckdb.org/docs/extensions/full_text_search)
- [VSS Extension](https://duckdb.org/docs/extensions/vss)
- [JSON Functions](https://duckdb.org/docs/extensions/json)

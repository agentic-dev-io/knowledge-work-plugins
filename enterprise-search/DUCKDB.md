# DuckDB Configuration for Enterprise Search Plugin

## Overview

This plugin uses DuckDB with full-text search and vector similarity capabilities to enable semantic search across enterprise content, documents, and communications.

## Database Location

```
~/knowledge-work-data/enterprise-search/enterprise-search.duckdb
```

## Key Extensions

### VSS (Vector Similarity Search)
Semantic search across all enterprise content.

```sql
INSTALL vss;
LOAD vss;

CREATE TABLE documents (
    doc_id INT,
    title VARCHAR,
    content TEXT,
    source VARCHAR,
    author VARCHAR,
    created_at TIMESTAMP,
    embedding FLOAT[768]
);

CREATE INDEX docs_idx ON documents 
USING HNSW (embedding) WITH (metric = 'cosine');

-- Semantic search
SELECT doc_id, title, source,
       array_cosine_similarity(embedding, $query::FLOAT[768]) as relevance
FROM documents
ORDER BY relevance DESC LIMIT 20;
```

### Full-Text Search
Keyword-based search with ranking.

```sql
INSTALL fts;
LOAD fts;

PRAGMA create_fts_index('documents', 'doc_id', 'title', 'content');

-- Combined keyword + semantic search
WITH keyword_results AS (
    SELECT doc_id, fts_main_documents.match_bm25(doc_id, 'quarterly business review') AS bm25_score
    FROM documents
),
semantic_results AS (
    SELECT doc_id, array_cosine_similarity(embedding, $query_embedding::FLOAT[768]) as semantic_score
    FROM documents
)
SELECT 
    d.*,
    COALESCE(k.bm25_score, 0) * 0.4 + COALESCE(s.semantic_score, 0) * 0.6 as final_score
FROM documents d
LEFT JOIN keyword_results k ON d.doc_id = k.doc_id
LEFT JOIN semantic_results s ON d.doc_id = s.doc_id
WHERE final_score > 0.3
ORDER BY final_score DESC;
```

## Common Search Queries

### Cross-Source Search
```sql
-- Search across multiple sources
SELECT 
    source,
    COUNT(*) as result_count,
    AVG(relevance_score) as avg_relevance
FROM search_results
WHERE query = 'product roadmap Q4'
GROUP BY source
ORDER BY avg_relevance DESC;
```

### Usage Analytics
```sql
-- Track search patterns
SELECT 
    DATE_TRUNC('day', search_timestamp) as date,
    COUNT(DISTINCT user_id) as unique_searchers,
    COUNT(*) as total_searches,
    AVG(result_count) as avg_results,
    SUM(CASE WHEN clicked_result THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as click_through_rate
FROM search_logs
GROUP BY date
ORDER BY date DESC;
```

## Recommended Additional Extensions (Priority P1/P2)

### Advanced Fuzzy Search

#### Rapidfuzz Extension
Fuzzy search across all enterprise platforms.

```sql
INSTALL rapidfuzz;
LOAD rapidfuzz;

-- Fuzzy search across documents
SELECT 
    doc_id,
    title,
    source,
    rapidfuzz_similarity(title, $search_query) as title_match,
    rapidfuzz_partial_ratio(content, $search_query) as content_match,
    (title_match * 0.7 + content_match * 0.3) as combined_score
FROM documents
WHERE combined_score > 60
ORDER BY combined_score DESC;

-- Typo-tolerant search
SELECT 
    document_id,
    filename,
    rapidfuzz_token_set_ratio(filename, $fuzzy_query) as match_score
FROM files
WHERE rapidfuzz_token_set_ratio(filename, $fuzzy_query) > 75;
```

#### Fuzzycomplete Extension
Auto-complete for enterprise search queries.

```sql
INSTALL fuzzycomplete;
LOAD fuzzycomplete;

-- Search term auto-completion
SELECT 
    search_term,
    search_count,
    fuzzy_complete_score(search_term, $partial_input) as completion_score
FROM popular_searches
WHERE fuzzy_complete_score(search_term, $partial_input) > 0.6
ORDER BY completion_score DESC, search_count DESC
LIMIT 10;

-- Smart file path completion
SELECT 
    file_path,
    fuzzy_path_complete(file_path, $partial_path) as path_score
FROM indexed_files
WHERE fuzzy_path_complete(file_path, $partial_path) > 0.7
ORDER BY path_score DESC;
```

### Fast String Lookups

#### Marisa Extension
Trie-based fast lookups for document indexing.

```sql
INSTALL marisa;
LOAD marisa;

-- Create trie index for document titles
CREATE MARISA TRIE doc_title_trie ON documents(title);

-- Fast prefix search
SELECT doc_id, title, source, created_at
FROM documents
WHERE marisa_prefix_search(title, $prefix)
ORDER BY created_at DESC
LIMIT 20;

-- Efficient autocomplete for tags
CREATE MARISA TRIE tag_trie ON document_tags(tag_name);

SELECT DISTINCT tag_name, count(*) as usage_count
FROM document_tags
WHERE marisa_prefix_search(tag_name, $tag_prefix)
GROUP BY tag_name
ORDER BY usage_count DESC;
```

### Content Processing

#### Webbed Extension
Parse HTML/XML documents and wikis.

```sql
INSTALL webbed;
LOAD webbed;

-- Extract content from HTML docs
SELECT 
    doc_id,
    url,
    xpath(html_content, '//title') as page_title,
    xpath(html_content, '//main//p') as main_content,
    extract_links(html_content) as internal_links
FROM html_documents;

-- Parse wiki pages
SELECT 
    wiki_page_id,
    extract_headings(html) as page_structure,
    xpath(html, '//div[@class="toc"]//li') as table_of_contents
FROM wiki_pages;
```

#### Markdown Extension
Index and search markdown documentation.

```sql
INSTALL markdown;
LOAD markdown;

-- Index markdown documents
SELECT 
    doc_id,
    filename,
    extract_markdown_headers(content, 1) as h1_sections,
    extract_markdown_headers(content, 2) as h2_sections,
    extract_markdown_code_blocks(content) as code_blocks,
    markdown_to_plaintext(content) as searchable_text
FROM markdown_docs;

-- Search code examples in docs
SELECT 
    doc_id,
    title,
    extract_markdown_code_blocks(content, 'sql') as sql_examples
FROM technical_docs
WHERE array_length(extract_markdown_code_blocks(content, 'sql')) > 0;
```

### Session Management

#### TSID Extension
Generate unique IDs for search sessions and tracking.

```sql
INSTALL tsid;
LOAD tsid;

-- Track search sessions
INSERT INTO search_sessions (session_id, user_id, query, timestamp)
VALUES (tsid(), $user_id, $query, now());

-- Analyze search patterns by session
SELECT 
    session_id,
    user_id,
    count(*) as query_count,
    array_agg(query ORDER BY timestamp) as query_sequence
FROM search_sessions
GROUP BY session_id, user_id
HAVING count(*) > 1;
```

## Extension Priority Matrix

| Priority | Extensions | Purpose |
|----------|-----------|---------|
| **P1 (High)** | rapidfuzz, fuzzycomplete | Fuzzy search & auto-complete |
| **P1 (High)** | marisa | Fast trie-based lookups |
| **P1 (High)** | webbed, markdown | Document parsing |
| **P2 (Medium)** | tsid | Session tracking |

## Quick Start with Enterprise Search Extensions

```sql
-- Install all recommended extensions
INSTALL rapidfuzz;
INSTALL fuzzycomplete;
INSTALL marisa;
INSTALL webbed;
INSTALL markdown;
INSTALL tsid;

-- Load for current session
LOAD rapidfuzz;
LOAD fuzzycomplete;
LOAD marisa;
LOAD webbed;
LOAD markdown;
LOAD tsid;
```

## Resources

- [DuckDB Full-Text Search](https://duckdb.org/docs/extensions/full_text_search)
- [VSS Extension](https://duckdb.org/docs/extensions/vss)

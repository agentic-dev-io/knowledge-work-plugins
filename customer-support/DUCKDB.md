# DuckDB Configuration for Customer Support Plugin

## Overview

This plugin uses DuckDB for support ticket analysis, customer interaction tracking, and knowledge base management with full-text and semantic search capabilities.

## Database Location

```
~/knowledge-work-data/customer-support/customer-support.duckdb
```

## Key Extensions

### VSS (Vector Similarity Search)
Find similar tickets, suggest solutions, and match customer issues to knowledge base articles.

```sql
INSTALL vss;
LOAD vss;

-- Store ticket embeddings
CREATE TABLE support_tickets (
    ticket_id INT,
    customer_id INT,
    subject VARCHAR,
    description TEXT,
    status VARCHAR,
    priority VARCHAR,
    embedding FLOAT[384]
);

CREATE INDEX tickets_idx ON support_tickets 
USING HNSW (embedding) WITH (metric = 'cosine');

-- Find similar tickets
SELECT ticket_id, subject, status,
       array_cosine_similarity(embedding, $query_embedding::FLOAT[384]) as similarity
FROM support_tickets
WHERE status = 'resolved'
ORDER BY similarity DESC LIMIT 5;
```

### Full-Text Search
Search through tickets, knowledge base, and customer interactions.

```sql
INSTALL fts;
LOAD fts;

-- Index knowledge base articles
PRAGMA create_fts_index('kb_articles', 'article_id', 'title', 'content');

-- Search for solutions
SELECT article_id, title, score
FROM (
    SELECT *, fts_main_kb_articles.match_bm25(article_id, 'password reset email') AS score
    FROM kb_articles
) WHERE score IS NOT NULL ORDER BY score DESC;
```

## Common Support Queries

### Ticket Volume and Response Time Analysis
```sql
-- Daily ticket metrics
SELECT 
    DATE_TRUNC('day', created_at) as date,
    COUNT(*) as tickets_created,
    AVG(EXTRACT(EPOCH FROM (first_response_at - created_at)) / 3600) as avg_response_hours,
    AVG(EXTRACT(EPOCH FROM (resolved_at - created_at)) / 3600) as avg_resolution_hours,
    SUM(CASE WHEN priority = 'high' THEN 1 ELSE 0 END) as high_priority
FROM support_tickets
WHERE created_at >= CURRENT_DATE - INTERVAL 30 DAY
GROUP BY date
ORDER BY date DESC;
```

### Customer Support History
```sql
-- Customer interaction timeline
SELECT 
    customer_id,
    COUNT(*) as total_tickets,
    SUM(CASE WHEN status = 'resolved' THEN 1 ELSE 0 END) as resolved_tickets,
    AVG(satisfaction_score) as avg_satisfaction,
    MAX(created_at) as last_ticket_date,
    STRING_AGG(DISTINCT category, ', ') as common_issues
FROM support_tickets
GROUP BY customer_id
HAVING total_tickets >= 3
ORDER BY total_tickets DESC;
```

### Escalation Pattern Analysis
```sql
-- Track escalations
SELECT 
    category,
    COUNT(*) as total_tickets,
    SUM(CASE WHEN escalated = TRUE THEN 1 ELSE 0 END) as escalated,
    SUM(CASE WHEN escalated = TRUE THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as escalation_rate,
    AVG(CASE WHEN escalated = TRUE THEN resolution_hours END) as avg_escalated_resolution
FROM support_tickets
GROUP BY category
ORDER BY escalation_rate DESC;
```

## Recommended Additional Extensions (Priority P1/P2)

### Fuzzy Matching & Similar Ticket Detection

#### Rapidfuzz Extension
Find similar support tickets for solution suggestions.

```sql
INSTALL rapidfuzz;
LOAD rapidfuzz;

-- Find similar resolved tickets
SELECT 
    t1.ticket_id as new_ticket,
    t1.description as new_description,
    t2.ticket_id as similar_ticket,
    t2.resolution,
    rapidfuzz_similarity(t1.description, t2.description) as similarity_score
FROM tickets t1
CROSS JOIN tickets t2
WHERE t1.status = 'open'
  AND t2.status = 'resolved'
  AND rapidfuzz_similarity(t1.description, t2.description) > 0.75
ORDER BY similarity_score DESC
LIMIT 5;

-- Match tickets to KB articles
SELECT 
    ticket_id,
    ticket_subject,
    kb_article_id,
    kb_title,
    rapidfuzz_token_sort_ratio(ticket_subject, kb_title) as relevance
FROM tickets, kb_articles
WHERE rapidfuzz_token_sort_ratio(ticket_subject, kb_title) > 70
ORDER BY relevance DESC;
```

#### Splink UDFs Extension
Advanced probabilistic record linkage for customer deduplication.

```sql
INSTALL splink_udfs;
LOAD splink_udfs;

-- Deduplicate customer records with probabilistic matching
SELECT 
    c1.customer_id,
    c2.customer_id as potential_duplicate,
    splink_jaro_winkler(c1.name, c2.name) as name_similarity,
    splink_levenshtein(c1.email, c2.email) as email_distance,
    splink_match_probability(
        c1.name, c2.name,
        c1.email, c2.email,
        c1.phone, c2.phone
    ) as match_probability
FROM customers c1, customers c2
WHERE c1.customer_id < c2.customer_id
  AND splink_match_probability(c1.name, c2.name, c1.email, c2.email, c1.phone, c2.phone) > 0.8;
```

### Content Processing

#### Markdown Extension
Generate and process knowledge base articles.

```sql
INSTALL markdown;
LOAD markdown;

-- Convert resolved tickets to KB articles
SELECT 
    ticket_id,
    '# ' || subject || '\n\n' || 
    '## Problem\n\n' || description || '\n\n' ||
    '## Solution\n\n' || resolution as kb_article_markdown,
    markdown_to_html(kb_article_markdown) as kb_article_html
FROM resolved_tickets
WHERE resolution_quality_score > 8;

-- Parse existing KB articles
SELECT 
    article_id,
    extract_markdown_headers(content, 2) as sections,
    extract_markdown_code_blocks(content) as code_examples,
    markdown_word_count(content) as word_count
FROM kb_articles;
```

### Data Transformation

#### JSONata Extension
Combine customer data from multiple support tools.

```sql
INSTALL jsonata;
LOAD jsonata;

-- Transform Intercom conversation data
SELECT 
    conversation_id,
    jsonata_transform(intercom_data,
        '{
            "customer": user.name,
            "email": user.email,
            "messages": conversation_parts[].{
                "author": author.name,
                "text": body,
                "timestamp": created_at
            }
        }') as normalized_conversation
FROM intercom_conversations;

-- Aggregate support data from multiple channels
SELECT 
    customer_id,
    jsonata_transform(
        json_object(
            'zendesk_tickets', zendesk_data,
            'slack_messages', slack_data,
            'email_threads', email_data
        ),
        '{
            "total_interactions": $count(zendesk_tickets) + $count(slack_messages) + $count(email_threads),
            "channels": [$distinct([zendesk_tickets.channel, slack_messages.channel, "email"])],
            "sentiment": $average([zendesk_tickets.sentiment, slack_messages.sentiment])
        }'
    ) as customer_360
FROM customer_interactions;
```

#### Fuzzycomplete Extension
Auto-complete for standard support responses.

```sql
INSTALL fuzzycomplete;
LOAD fuzzycomplete;

-- Auto-complete canned responses
SELECT 
    response_id,
    response_title,
    response_text,
    fuzzy_complete_score(response_title, $partial_search) as match_score
FROM canned_responses
WHERE fuzzy_complete_score(response_title, $partial_search) > 0.6
ORDER BY match_score DESC
LIMIT 10;

-- Smart tag suggestions
SELECT 
    ticket_id,
    suggested_tag,
    fuzzy_complete_score(ticket_description, tag_keywords) as relevance
FROM tickets, tag_library
WHERE fuzzy_complete_score(ticket_description, tag_keywords) > 0.5
ORDER BY relevance DESC;
```

## Extension Priority Matrix

| Priority | Extensions | Purpose |
|----------|-----------|---------|
| **P1 (High)** | rapidfuzz | Similar ticket detection |
| **P1 (High)** | markdown | KB article generation |
| **P1 (High)** | jsonata | Multi-channel data aggregation |
| **P2 (Medium)** | splink_udfs | Advanced deduplication |
| **P2 (Medium)** | fuzzycomplete | Response auto-complete |

## Quick Start with Customer Support Extensions

```sql
-- Install all recommended extensions
INSTALL rapidfuzz;
INSTALL splink_udfs;
INSTALL markdown;
INSTALL jsonata;
INSTALL fuzzycomplete;

-- Load for current session
LOAD rapidfuzz;
LOAD splink_udfs;
LOAD markdown;
LOAD jsonata;
LOAD fuzzycomplete;
```

## Resources

- [DuckDB Documentation](https://duckdb.org/docs/)
- [Full-Text Search](https://duckdb.org/docs/extensions/full_text_search)
- [VSS Extension](https://duckdb.org/docs/extensions/vss)

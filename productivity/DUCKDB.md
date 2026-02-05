# DuckDB Configuration for Productivity Plugin

## Overview

This plugin uses DuckDB for personal task management, calendar analytics, and productivity tracking across multiple work tools.

## Database Location

```
~/knowledge-work-data/productivity/productivity.duckdb
```

## Key Extensions

### VSS (Vector Similarity Search)
Find related tasks, similar meeting topics, and relevant documents.

```sql
INSTALL vss;
LOAD vss;

CREATE TABLE tasks (
    task_id INT,
    title VARCHAR,
    description TEXT,
    status VARCHAR,
    priority VARCHAR,
    due_date DATE,
    embedding FLOAT[384]
);

CREATE INDEX tasks_idx ON tasks 
USING HNSW (embedding) WITH (metric = 'cosine');

-- Find related tasks
SELECT task_id, title, status,
       array_cosine_similarity(embedding, $current_task::FLOAT[384]) as similarity
FROM tasks
WHERE status = 'completed'
ORDER BY similarity DESC LIMIT 5;
```

### Full-Text Search
Search across tasks, notes, and meeting transcripts.

```sql
INSTALL fts;
LOAD fts;

PRAGMA create_fts_index('notes', 'note_id', 'title', 'content');

SELECT * FROM (
    SELECT *, fts_main_notes.match_bm25(note_id, 'project planning') AS score
    FROM notes
) WHERE score IS NOT NULL ORDER BY score DESC;
```

## Common Productivity Queries

### Task Completion Analytics
```sql
-- Track completion rates over time
SELECT 
    DATE_TRUNC('week', completed_at) as week,
    COUNT(*) as tasks_completed,
    AVG(EXTRACT(DAY FROM (completed_at - created_at))) as avg_completion_days,
    SUM(CASE WHEN completed_at <= due_date THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as on_time_rate
FROM tasks
WHERE status = 'completed'
GROUP BY week
ORDER BY week DESC;
```

### Calendar Time Analysis
```sql
-- Meeting time breakdown
SELECT 
    DATE_TRUNC('week', meeting_date) as week,
    SUM(duration_minutes) / 60.0 as total_meeting_hours,
    COUNT(*) as meeting_count,
    AVG(duration_minutes) as avg_meeting_length,
    SUM(CASE WHEN attendee_count > 5 THEN duration_minutes ELSE 0 END) / 60.0 as large_meeting_hours
FROM calendar_events
WHERE event_type = 'meeting'
GROUP BY week
ORDER BY week DESC;
```

### Focus Time Tracking
```sql
-- Identify focus time vs. fragmented time
WITH time_blocks AS (
    SELECT 
        DATE_TRUNC('day', date) as day,
        EXTRACT(HOUR FROM start_time) as hour,
        SUM(CASE WHEN event_type = 'meeting' THEN 1 ELSE 0 END) as meeting_count
    FROM calendar_events
    GROUP BY day, hour
)
SELECT 
    day,
    SUM(CASE WHEN meeting_count = 0 THEN 1 ELSE 0 END) as focus_hours,
    SUM(CASE WHEN meeting_count > 0 THEN 1 ELSE 0 END) as meeting_hours,
    SUM(CASE WHEN meeting_count = 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as focus_time_pct
FROM time_blocks
WHERE hour >= 9 AND hour < 18  -- Working hours
GROUP BY day
ORDER BY day DESC;
```

## Recommended Additional Extensions (Priority P2/P3)

### Data Transformation

#### JSONata Extension
Combine task data from multiple productivity tools.

```sql
INSTALL jsonata;
LOAD jsonata;

-- Transform tasks from different tools
SELECT 
    jsonata_transform(asana_tasks,
        'tasks[].{
            "id": gid,
            "title": name,
            "status": completed ? "done" : "todo",
            "due": due_on,
            "source": "asana"
        }') as normalized_tasks
FROM asana_api;

-- Aggregate calendar events
SELECT 
    jsonata_transform(calendar_data,
        'events[].{
            "title": summary,
            "start": start.dateTime,
            "duration": $duration(start.dateTime, end.dateTime),
            "attendees": $count(attendees)
        }') as processed_events
FROM google_calendar;
```

#### YAML Extension
Parse and manage workflow configurations.

```sql
INSTALL yaml;
LOAD yaml;

-- Load workflow automation config
SELECT 
    workflow_name,
    yaml_extract(config, '$.triggers') as triggers,
    yaml_extract(config, '$.actions') as actions,
    yaml_extract(config, '$.conditions') as conditions
FROM workflow_configs;

-- Parse project templates
SELECT 
    template_name,
    yaml_extract(template, '$.tasks') as task_list,
    yaml_extract(template, '$.milestones') as milestones
FROM project_templates;
```

### Approximate Analytics

#### DataSketches Extension
Efficient approximate counts for task statistics.

```sql
INSTALL datasketches;
LOAD datasketches;

-- Approximate unique users per project (memory efficient)
SELECT 
    project_id,
    theta_sketch_distinct(user_id) as approx_unique_contributors,
    theta_sketch_estimate(user_id) as estimated_count
FROM task_activity
GROUP BY project_id;

-- HyperLogLog for large-scale task metrics
SELECT 
    week,
    hll_sketch_count(distinct task_id) as tasks_touched,
    hll_sketch_count(distinct user_id) as active_users
FROM weekly_activity
GROUP BY week;
```

## Extension Priority Matrix

| Priority | Extensions | Purpose |
|----------|-----------|---------|
| **P2 (Medium)** | jsonata | Multi-tool task aggregation |
| **P2 (Medium)** | yaml | Workflow configuration |
| **P3 (Nice-to-have)** | datasketches | Task statistics (approximate) |

## Quick Start with Productivity Extensions

```sql
-- Install recommended extensions
INSTALL jsonata;
INSTALL yaml;
INSTALL datasketches;

-- Load for current session
LOAD jsonata;
LOAD yaml;
LOAD datasketches;
```

## Resources

- [DuckDB Date Functions](https://duckdb.org/docs/sql/functions/date)
- [Window Functions](https://duckdb.org/docs/sql/window_functions)

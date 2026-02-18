---
name: data-exploration
description: Profile and explore datasets to understand their shape, quality, and patterns before analysis. Use when encountering a new dataset, assessing data quality, discovering column distributions, identifying nulls and outliers, or deciding which dimensions to analyze.
---

# Data Exploration Skill

Systematic methodology for profiling datasets, assessing data quality, discovering patterns, and understanding schemas.

## Data Profiling Methodology

### Phase 1: Structural Understanding

Before analyzing any data, understand its structure:

**Table-level questions:**
- How many rows and columns?
- What is the grain (one row per what)?
- What is the primary key? Is it unique?
- When was the data last updated?
- How far back does the data go?

**Column classification:**
Categorize each column as one of:
- **Identifier**: Unique keys, foreign keys, entity IDs
- **Dimension**: Categorical attributes for grouping/filtering (status, type, region, category)
- **Metric**: Quantitative values for measurement (revenue, count, duration, score)
- **Temporal**: Dates and timestamps (created_at, updated_at, event_date)
- **Text**: Free-form text fields (description, notes, name)
- **Boolean**: True/false flags
- **Structural**: JSON, arrays, nested structures

### Phase 2: Column-Level Profiling

For each column, compute:

**All columns:**
- Null count and null rate
- Distinct count and cardinality ratio (distinct / total)
- Most common values (top 5-10 with frequencies)
- Least common values (bottom 5 to spot anomalies)

**Numeric columns (metrics):**
```
min, max, mean, median (p50)
standard deviation
percentiles: p1, p5, p25, p75, p95, p99
zero count
negative count (if unexpected)
```

**String columns (dimensions, text):**
```
min length, max length, avg length
empty string count
pattern analysis (do values follow a format?)
case consistency (all upper, all lower, mixed?)
leading/trailing whitespace count
```

**Date/timestamp columns:**
```
min date, max date
null dates
future dates (if unexpected)
distribution by month/week
gaps in time series
```

**Boolean columns:**
```
true count, false count, null count
true rate
```

### Phase 3: Relationship Discovery

After profiling individual columns:

- **Foreign key candidates**: ID columns that might link to other tables
- **Hierarchies**: Columns that form natural drill-down paths (country > state > city)
- **Correlations**: Numeric columns that move together
- **Derived columns**: Columns that appear to be computed from others
- **Redundant columns**: Columns with identical or near-identical information

## Quality Assessment Framework

### Completeness Score

Rate each column:
- **Complete** (>99% non-null): Green
- **Mostly complete** (95-99%): Yellow -- investigate the nulls
- **Incomplete** (80-95%): Orange -- understand why and whether it matters
- **Sparse** (<80%): Red -- may not be usable without imputation

### Consistency Checks

Look for:
- **Value format inconsistency**: Same concept represented differently ("USA", "US", "United States", "us")
- **Type inconsistency**: Numbers stored as strings, dates in various formats
- **Referential integrity**: Foreign keys that don't match any parent record
- **Business rule violations**: Negative quantities, end dates before start dates, percentages > 100
- **Cross-column consistency**: Status = "completed" but completed_at is null

### Accuracy Indicators

Red flags that suggest accuracy issues:
- **Placeholder values**: 0, -1, 999999, "N/A", "TBD", "test", "xxx"
- **Default values**: Suspiciously high frequency of a single value
- **Stale data**: Updated_at shows no recent changes in an active system
- **Impossible values**: Ages > 150, dates in the far future, negative durations
- **Round number bias**: All values ending in 0 or 5 (suggests estimation, not measurement)

### Timeliness Assessment

- When was the table last updated?
- What is the expected update frequency?
- Is there a lag between event time and load time?
- Are there gaps in the time series?

## Pattern Discovery Techniques

### Distribution Analysis

For numeric columns, characterize the distribution:
- **Normal**: Mean and median are close, bell-shaped
- **Skewed right**: Long tail of high values (common for revenue, session duration)
- **Skewed left**: Long tail of low values (less common)
- **Bimodal**: Two peaks (suggests two distinct populations)
- **Power law**: Few very large values, many small ones (common for user activity)
- **Uniform**: Roughly equal frequency across range (often synthetic or random)

### Temporal Patterns

For time series data, look for:
- **Trend**: Sustained upward or downward movement
- **Seasonality**: Repeating patterns (weekly, monthly, quarterly, annual)
- **Day-of-week effects**: Weekday vs. weekend differences
- **Holiday effects**: Drops or spikes around known holidays
- **Change points**: Sudden shifts in level or trend
- **Anomalies**: Individual data points that break the pattern

### Segmentation Discovery

Identify natural segments by:
- Finding categorical columns with 3-20 distinct values
- Comparing metric distributions across segment values
- Looking for segments with significantly different behavior
- Testing whether segments are homogeneous or contain sub-segments

### Correlation Exploration

Between numeric columns:
- Compute correlation matrix for all metric pairs
- Flag strong correlations (|r| > 0.7) for investigation
- Note: Correlation does not imply causation -- flag this explicitly
- Check for non-linear relationships (e.g., quadratic, logarithmic)

## Schema Understanding and Documentation

### Schema Documentation Template

When documenting a dataset for team use:

```markdown
## Table: [schema.table_name]

**Description**: [What this table represents]
**Grain**: [One row per...]
**Primary Key**: [column(s)]
**Row Count**: [approximate, with date]
**Update Frequency**: [real-time / hourly / daily / weekly]
**Owner**: [team or person responsible]

### Key Columns

| Column | Type | Description | Example Values | Notes |
|--------|------|-------------|----------------|-------|
| user_id | STRING | Unique user identifier | "usr_abc123" | FK to users.id |
| event_type | STRING | Type of event | "click", "view", "purchase" | 15 distinct values |
| revenue | DECIMAL | Transaction revenue in USD | 29.99, 149.00 | Null for non-purchase events |
| created_at | TIMESTAMP | When the event occurred | 2024-01-15 14:23:01 | Partitioned on this column |

### Relationships
- Joins to `users` on `user_id`
- Joins to `products` on `product_id`
- Parent of `event_details` (1:many on event_id)

### Known Issues
- [List any known data quality issues]
- [Note any gotchas for analysts]

### Common Query Patterns
- [Typical use cases for this table]
```

### Schema Exploration Queries

When connected to a data warehouse, use these patterns to discover schema:

```sql
-- List all tables in a schema (PostgreSQL)
SELECT table_name, table_type
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name;

-- Column details (PostgreSQL)
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = 'my_table'
ORDER BY ordinal_position;

-- Table sizes (PostgreSQL)
SELECT relname, pg_size_pretty(pg_total_relation_size(relid))
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Row counts for all tables (general pattern)
-- Run per-table: SELECT COUNT(*) FROM table_name
```

### Lineage and Dependencies

When exploring an unfamiliar data environment:

1. Start with the "output" tables (what reports or dashboards consume)
2. Trace upstream: What tables feed into them?
3. Identify raw/staging/mart layers
4. Map the transformation chain from raw data to analytical tables
5. Note where data is enriched, filtered, or aggregated

---

## Streaming & Legacy Enterprise Data Sources

### Streaming Sources (Kafka and Event Streams)

Streaming data requires a different exploration posture — you cannot do a simple `COUNT(*)` or full scan. Instead, sample and profile over a time window.

**Key questions for streaming data:**
- What is the topic name, partition count, and replication factor?
- What is the message format (Avro, Protobuf, JSON, raw bytes)?
- Where is the schema stored? (Schema Registry, inline, or undocumented?)
- What is the approximate message rate (msgs/sec) and retention period?
- Is there a consumer group already reading this topic? What is the consumer lag?
- Are messages keyed? What determines the key, and does it imply ordering guarantees?

**Sampling strategy for Kafka topics:**
```bash
# Sample recent messages (Kafka CLI)
kafka-console-consumer --topic <topic> \
  --bootstrap-server <broker> \
  --from-beginning \
  --max-messages 500 \
  --timeout-ms 10000

# Check topic metadata and partition offsets
kafka-topics --describe --topic <topic> --bootstrap-server <broker>
kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list <broker> --topic <topic>
```

**Profiling checklist for event streams:**
- Sample at least 500–1000 messages from different partitions and time windows
- Identify the **event schema**: Are all messages the same shape, or is the topic a "firehose" of mixed event types?
- Check for **schema evolution**: Do older messages have different fields than newer ones?
- Measure **timestamp skew**: Difference between event time (in payload) and Kafka ingestion time
- Identify **duplicates**: Does the stream provide at-least-once delivery? Is there a deduplication key?
- Look for **null or missing fields** that vary by event type
- Note whether the stream is **append-only** or includes updates/deletes (e.g., CDC streams)

**Structural classification for stream fields** (extends standard column classification):
- **Event envelope fields**: `event_type`, `event_id`, `source_system`, `schema_version` — these are infrastructure fields, not business fields
- **Entity keys**: IDs that link the event to a business entity (order_id, user_id)
- **State fields**: Values that represent a snapshot of state at event time (order_status, cart_total)
- **Delta fields**: Values that represent a change (amount_charged, items_added)

---

### Legacy Enterprise Sources

Large enterprises commonly have data locked in systems that predate modern data platforms. Treat these with extra scrutiny around completeness, encoding, and undocumented conventions.

#### Relational Legacy Systems (Oracle, DB2, Teradata, Sybase)

- **Watch for proprietary data types**: `NUMBER(38,0)` used as booleans, `CHAR` with trailing spaces, `DATE` types that include time components
- **Codified values are common**: Columns storing `'Y'`/`'N'`, `1`/`0`/`2`, or cryptic codes (`'STS_03'`) — always look for a reference/lookup table
- **Sequences and surrogate keys** often have gaps; don't assume they are contiguous
- **Soft deletes**: Look for `is_deleted`, `active_flag`, `record_status`, `expiry_date` — failing to filter these is a frequent source of double-counting
- **Effective-dated records (SCD)**: Many legacy systems use `effective_start_date` / `effective_end_date` for history; confirm whether the table is current-state-only or full history

```sql
-- Check for soft-delete patterns
SELECT is_deleted, COUNT(*) FROM my_table GROUP BY is_deleted;

-- Check for SCD2 / effective dating
SELECT MIN(effective_start_date), MAX(effective_end_date),
       COUNT(*), COUNT(DISTINCT entity_id)
FROM my_table;

-- Spot codified columns (low cardinality strings)
SELECT status_code, COUNT(*) FROM my_table GROUP BY status_code ORDER BY 2 DESC;
```

#### Mainframe / COBOL Sources (VSAM, flat files, copybooks)

- Data arrives as fixed-width flat files or extracted CSV dumps — confirm the **encoding** (EBCDIC vs UTF-8) and **line ending** format
- Numeric fields may be **packed decimal** or **zoned decimal** — verify the extraction tool handled conversion correctly (garbled numerics are common)
- **REDEFINES clauses** in COBOL copybooks mean a single physical field can represent different things depending on a condition flag elsewhere in the record — ask for the copybook
- Date fields are frequently stored as `YYYYMMDD` integers or 2-digit years — check for Y2K-era pivoting logic (`00–49` → 2000s, `50–99` → 1900s)
- Verify record counts from the extract against the source system control totals if available

#### ERP Systems (SAP, Oracle EBS, PeopleSoft)

- These systems have **extremely wide tables** (hundreds of columns); most columns are null or unused for a given client configuration — run null-rate profiling first to discard empty columns
- **Client/org/company code** columns partition data by legal entity — always confirm which codes are in scope
- SAP in particular uses **many join tables** (e.g., BKPF + BSEG for FI documents) — a single business concept may require 3–5 joins to reconstruct
- Posting dates, document dates, and value dates are distinct and matter differently for different analyses — clarify with the business owner
- Archived or "statistically posted" records may appear in extracts but should be excluded from most analyses

#### API / File-Based Feeds (SFTP drops, EDI, XML/JSON exports)

- Confirm the **feed SLA**: how often does the file arrive, and what is the cutoff time?
- Check for **partial files**: compare record counts to prior drops; a file 30% smaller than usual is a red flag
- Identify the **delta strategy**: is each file a full snapshot or an incremental delta? If delta, how are deletes communicated?
- For XML/JSON: check for **nested arrays** that need unnesting — they are a frequent source of fan-out and row multiplication bugs
- Validate **control totals or checksums** if the feed provides them

---

### Streaming + Legacy Combined: CDC Streams

Change Data Capture (CDC) pipelines (Debezium, Oracle GoldenGate, IBM InfoSphere) emit legacy database changes as Kafka events. These combine both worlds:

- Each message has an **operation type**: `INSERT`, `UPDATE`, `DELETE` (or `c`, `u`, `d` in Debezium)
- Messages include **before and after images** of the row — profile both
- **Schema changes** in the source table appear as special metadata events — check if the pipeline handles these gracefully
- To reconstruct current state, you must **compact** the stream by entity key, keeping only the latest non-DELETE event
- Confirm **transaction boundaries**: are multi-row transactions guaranteed to arrive atomically or can they be interleaved?

---

### Source-Specific Profiling Additions to Phase 1

Extend the structural understanding phase with these questions for enterprise sources:

| Source Type | Additional Questions |
|---|---|
| Kafka topic | Message format? Schema registry? Avg lag? Retention period? |
| Legacy RDBMS | Soft-delete columns? SCD history? Codified lookup tables? |
| Mainframe extract | Encoding confirmed? Control totals available? Copybook available? |
| ERP export | Which client/org codes? Full snapshot or delta? Posting date vs. doc date? |
| File feed | Feed frequency? Full vs. incremental? Control total available? |
| CDC stream | Source system? Operation field name? Before/after images present? |

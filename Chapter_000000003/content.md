# Chapter 3: File Formats: Choosing Your Poison

## Or: How JSON Became Everyone's Problem and Nobody's Solution

**In 2020, the mpv media player discovered that YouTube's API would sometimes return the string "no" instead of null for missing values - but only for Norwegian users. Why? Because "no" is the Norwegian language code, and somewhere deep in Google's stack, a helpful library was "localizing" null values.**

Chapter 2 was about data types - the conceptual lie that a ZIP code is a number. This chapter is about file formats - the *mechanical* lies that determine how data gets stored, moved, and inevitably corrupted.

File formats are the shipping containers of the data world. You can have perfect cargo, but put it in the wrong container and it arrives damaged. Put it in the right container with the wrong manifest and nobody knows what they've got. The best data in the world is worthless if it's stored in a format that lies about its contents.

## 3.1 JSON: The Accidental Standard That Ate the World

When Douglas Crockford formalized JSON in 2001, he intended to create a "lightweight data-interchange format" based on a subset of JavaScript. He succeeded beyond anyone's wildest dreams, accidentally creating a format that would become the backbone of modern web APIs despite being fundamentally broken in ways that haunt developers daily.

The irony? JSON was supposed to be the simple alternative to XML. Crockford's design philosophy was to be minimal, portable, textual, and a subset of JavaScript. What we got instead was a format that every programming language interprets slightly differently.

### The Good, The Bad, and The Utterly Bizarre

**The Good:**
- Human-readable: You can debug with your eyeballs
- Widely supported: Every language since 2005 can parse it (badly)
- Flexible: Schema-optional means you can shove anything in there
- Simple syntax: Only six data types to misunderstand

**The Bad:**
- No schema enforcement: Every field is a surprise
- Type ambiguity: Numbers, strings, who's counting?
- Size inefficient: 70% of your payload might be quote marks
- No comments: Hope you didn't want documentation

**The Ugly:**

```json
{
  "price": "5.00",
  "price2": 5.00,
  "price3": "$5.00",
  "price4": null,
  "price5": "NaN",
  "price6": false,
  "price7": "null",
  "price8": "",
  "price9": "false"
}
```

Which of these is the actual price? Trick question - they're ALL prices from real APIs I've worked with. One system considered `"false"` to mean "price not yet determined." That string `"false"` is truthy in most programming languages, by the way.

### The Enterprise Anti-Pattern Hall of Fame

The Norway example in the opener is real, but it's just the tip of the iceberg. Here's what real-world JSON looks like:

```json
{
  "success": true,
  "data": {
    "success": "false",
    "results": {
      "data": [
        {
          "datum": {
            "information": {
              "data": "actual data here finally",
              "metadata": {
                "meta": {
                  "data_about_data": {
                    "is_this_data": "yes"
                  }
                }
              }
            }
          }
        }
      ]
    }
  },
  "status": "SUCCESS",
  "error": null,
  "errors": [],
  "errorMessage": "",
  "hasErrors": false,
  "is_error": "NO",
  "ERROR_CODE": 0,
  "errCode": "SUCCESS"
}
```

Developer: "It's self-documenting!"
Me: "I'm self-immolating."

### The JavaScript Number Disaster

Perhaps JSON's greatest sin is inheriting JavaScript's IEEE 754 floating-point numbers:

```javascript
// In JavaScript (JSON's birthplace)
JSON.parse('{"value": 9007199254740993}')  // Returns: 9007199254740992 (off by 1!)

// Why? JavaScript can't represent integers larger than 2^53-1 accurately
Number.MAX_SAFE_INTEGER  // 9007199254740991

// This breaks Twitter IDs, database primary keys, and cryptocurrency values
JSON.parse('{"bitcoin_satoshis": 2100000000000000}')  // Loss of precision!

// The "solution" many APIs use
{
  "id": 12345678901234567890,
  "id_str": "12345678901234567890"
}
```

Twitter famously ships both `id` (number) and `id_str` (string) for exactly this reason. Every API that deals with large integers eventually learns this lesson.

### The MongoDB Date Apocalypse

MongoDB's extended JSON created multiple competing date formats:

```json
// MongoDB Extended JSON v1
{"$date": "2024-01-01T00:00:00.000Z"}

// MongoDB Extended JSON v2 Canonical
{"$date": {"$numberLong": "1704067200000"}}

// MongoDB Extended JSON v2 Relaxed
{"$date": "2024-01-01T00:00:00.000Z"}

// What actually gets stored sometimes
{"$date": 1704067200000}

// What some drivers return
{"date": "Mon Jan 01 2024 00:00:00 GMT+0000 (UTC)"}
```

Good luck writing a parser that handles all of these. And good luck when someone switches MongoDB versions mid-project.

### Surviving JSON in Production

**Always use ISO 8601 dates with timezones:**
```json
{"created": "2024-01-15T10:30:00Z"}
```

Not this:
```json
{"created": "01/15/2024"}
{"created": 1705318200}
{"created": "Monday, January 15, 2024"}
```

**Use strings for large numbers:**
```json
{
  "id": "12345678901234567890",
  "amount_cents": "100000000000000000"
}
```

**Version your APIs explicitly:**
```json
{
  "api_version": "2024-01-15",
  "data": {}
}
```

**Consider JSON streaming for large datasets:**
```python
import ijson

# Stream parse large JSON files instead of loading into memory
parser = ijson.items(open('huge.json', 'rb'), 'results.item')
for item in parser:
    process(item)  # Process one item at a time
```

### The Paradox of JSON's Success

JSON succeeded not despite its flaws but because of them. Its looseness allows gradual API evolution without breaking clients, human debugging without special tools, universal language support without complex libraries, and flexibility that rigid formats can't match.

As Martin Kleppmann notes in "Designing Data-Intensive Applications," JSON's popularity is evidence that ease of use matters more than efficiency for many applications. The format won because it shipped, not because it was perfect.

## 3.2 Parquet: When You Need Speed and Have Trust Issues

In 2012, Netflix engineers watched their data scientists try to load CSV files into memory. Then they watched their servers catch fire. Then they built Parquet.

Apache Parquet emerged in 2013 from a collaboration between Twitter and Cloudera engineers who were tired of exactly this problem. The format was inspired by Google's Dremel paper and represents what happens when database people get tired of data scientists using CSVs.

### The Problem with CSV

CSV has been the cockroach of data formats since RFC 4180 tried to standardize it in 2005 (yes, CSV existed for decades before anyone tried to formally define it). Despite its problems, CSV persists because it's human-readable and universally supported. But the costs are real:

```python
# CSV approach (what not to do)
df = pd.read_csv('huge_file.csv')  # 5 minutes, 10GB RAM
df['number_column'].dtype  # object (aka string, because pandas gave up)
df['date_column'].dtype    # object (pandas: "is this a date? who knows!")
df.memory_usage(deep=True).sum() / 1024**3  # 10.2 GB

# Parquet approach (what your RAM will thank you for)
df = pd.read_parquet('huge_file.parquet')  # 5 seconds, 1GB RAM
df['number_column'].dtype  # int64 (because Parquet remembers)
df['date_column'].dtype    # datetime64[ns] (Parquet kept the metadata!)
df.memory_usage(deep=True).sum() / 1024**3  # 1.1 GB
```

### Why Parquet is 10x+ Better

**Columnar Storage:** Unlike row-based formats (CSV, JSON), Parquet stores data column by column. This seemingly simple change enables massive optimizations because similar values compress better together, and you only read the columns you need.

```python
import pandas as pd
import numpy as np

# Create sample data
df = pd.DataFrame({
    'user_id': np.repeat(np.arange(1000000), 10),
    'timestamp': pd.date_range('2024-01-01', periods=10000000, freq='1s'),
    'value': np.random.randn(10000000),
    'category': np.random.choice(['A', 'B', 'C', 'D'], 10000000)
})

# Save as CSV vs Parquet
df.to_csv('data.csv', index=False)     # 410 MB on disk
df.to_parquet('data.parquet')          # 38 MB on disk (11x smaller!)

# Reading performance
%timeit pd.read_csv('data.csv', usecols=['user_id', 'value'])  # 2.3 seconds
%timeit pd.read_parquet('data.parquet', columns=['user_id', 'value'])  # 0.08 seconds (29x faster!)
```

**Predicate Pushdown:** Parquet files contain metadata about each column chunk, including min/max values. Query engines can skip entire chunks without reading them:

```python
import pyarrow.parquet as pq

# Read only data from February (skips 11 months of data!)
feb_data = pq.read_table(
    'sensors.parquet',
    filters=[
        ('timestamp', '>=', pd.Timestamp('2024-02-01')),
        ('timestamp', '<', pd.Timestamp('2024-03-01'))
    ]
)
# Only reads ~8% of the file from disk
```

### Real-World Performance: The Numbers

**Netflix's Big Data Platform:**
- 7x reduction in storage costs
- 10-100x improvement in query performance
- 90% reduction in compute costs for ETL pipelines

**Uber's Data Lake:**
- Processing 100+ petabytes daily
- 5x faster Presto queries compared to ORC
- 60% less storage compared to JSON

### When to Use Parquet (and When Not To)

**Perfect Use Cases:**
- Analytics workloads (read specific columns across many rows)
- Data warehousing (write once, read many times)
- Large datasets (bigger than RAM)
- Type safety critical (financial data, scientific measurements)
- Cloud storage (minimize S3/GCS costs with compression)
- Spark/Dask/Ray workflows (native support)

**When NOT to Use Parquet:**
- Streaming appends (Parquet is immutable; use Avro instead)
- Row-based access (if you need full rows frequently)
- Text processing pipelines (Unix tools can't grep Parquet)
- Frequent schema changes (each change requires rewriting files)
- Small datasets (under 100MB, CSV is probably fine)
- Excel users (they can't double-click to open Parquet)

### Parquet Optimization Tips

```python
import pyarrow.parquet as pq

# 1. Choose the right compression (snappy is fast, zstd is smaller)
df.to_parquet('file.parquet', compression='zstd', compression_level=9)

# 2. Optimize row group size for your queries
df.to_parquet('file.parquet', row_group_size=50000)

# 3. Sort by frequently filtered columns
df_sorted = df.sort_values('timestamp')
df_sorted.to_parquet('sorted.parquet')  # Min/max statistics now enable efficient skipping

# 4. Partition large datasets
df.to_parquet(
    'partitioned_data',
    partition_cols=['year', 'month'],  # Creates year=2024/month=01/ structure
    engine='pyarrow'
)
```

## 3.3 Apache Arrow: From Conversion Hell to Zero-Copy Bliss

Before Apache Arrow, the data science ecosystem resembled a Tower of Babel where every tool spoke its own language. Pandas had one memory layout, Spark had another, R had a third, Julia a fourth. Sharing data between tools required expensive serialization and deserialization.

```python
# The old way: A cascade of conversions
data_spark = spark.read.parquet("data.parquet")
data_pandas = data_spark.toPandas()  # 10 minutes of conversion, copies all data
data_numpy = data_pandas.values      # More conversion, another copy
model.train(data_numpy)               # Finally! After multiple copies and transforms
```

Apache Arrow, first released in 2016, fixed this by establishing a standard in-memory columnar format that could be shared across languages and systems without copying or converting.

```python
# The Arrow way: Direct access, no copies
import pyarrow as pa
table = pa.parquet.read_table("data.parquet")
# Everything can read Arrow directly. No conversion. Magic.
```

### The Zero-Copy Revolution

Arrow's most important feature is zero-copy reads. When different processes or languages need to access the same data:

- **Memory mapping**: Data can be memory-mapped directly from disk
- **Shared memory**: Processes can share the same memory pages without copying
- **Language agnostic**: C++, Python, R, Java, Rust, and Go can all read the same memory layout

This is possible because Arrow defines not just a logical format, but the exact memory layout down to the byte level.

### Integration with Modern Tools

Today, Arrow has become the de facto standard for analytical workloads:

- **Pandas 2.0+**: Uses Arrow as an optional backend, offering 2-10x performance improvements
- **Polars**: Built entirely on Arrow from the ground up
- **DuckDB**: Queries Arrow tables directly without conversion
- **Apache Spark 3.0+**: Uses Arrow for efficient data exchange with Python
- **Ray**: Leverages Arrow for distributed data processing

### The Performance Impact

According to Apache Arrow benchmarks:
- **100x faster** data interchange between systems
- **50-80% memory reduction** through shared memory instead of copies
- **10-100x faster** serialization/deserialization compared to pickle or JSON

```python
# The Arrow advantage (real benchmark)
# Task: Transfer 10GB dataset between tools

# Old way: Through CSV
df.to_csv("temp.csv")  # 5 minutes
spark.read.csv("temp.csv")  # 5 minutes
# Total: 10 minutes, 2x memory usage

# New way: Through Arrow
arrow_table = pa.Table.from_pandas(df)  # 3 seconds
spark_df = spark.createDataFrame(arrow_table)  # 0 seconds (zero-copy!)
# Total: 3 seconds, no extra memory
```

## 3.4 The Format Wars: Lakehouse, Lance, and What's Coming

The industry has spent the last decade learning an expensive lesson: no single data format wins all battles. The future isn't about picking the perfect format - it's about systems that can work with all of them.

### Lakehouse Architecture: The Best of Both Worlds

Data warehouses (Snowflake, BigQuery) and data lakes (S3, ADLS) are merging into "lakehouses" that provide structured queries over unstructured storage.

**Why It's Happening:**
- Warehouses cost $50K+/month for many companies
- Lakes are cheap but require armies of engineers
- Companies were maintaining both, doubling complexity

**Real Numbers From Production:**

*Uber's Migration to Apache Hudi (2022):*
- Reduced storage costs by 90% (from $2M to $200K annually)
- Query performance improved 5x
- Eliminated 10,000 lines of ETL code

*Netflix's Iceberg Adoption (2021):*
- Handles 300 petabytes with 10 engineers (was 50)
- Reduced time-to-insight from days to minutes
- Saved $10M annually in infrastructure costs

### Lance: Vector Database Meets Columnar Storage

Lance combines columnar storage (like Parquet) with vector indexing for AI workloads. Every company now has embeddings (from OpenAI, Cohere, etc.), and storing vectors in Parquet requires separate indexing (Pinecone, Weaviate = $$$).

```python
# The problem Lance solves
# Old way: Two systems, double the cost
metadata_df.to_parquet("products.parquet")  # Product info
vector_db.upsert(embeddings)  # $2000/month for Pinecone

# New way: One format, one query
import lance
dataset = lance.write_dataset(
    df_with_embeddings,  # Tabular + vectors together
    "products.lance"
)
# 10x faster queries, 75% less storage, no separate vector DB
```

### The Polyglot Persistence Pattern

Different access patterns need different storage:

| Access Pattern | Optimal Storage | Suboptimal | Speed Difference |
|---------------|-----------------|------------|------------------|
| Point Lookup | Redis/DynamoDB | Parquet | 500x slower |
| Analytics Scan | Parquet/BigQuery | MongoDB | 60x slower |
| Time Series | InfluxDB/TimescaleDB | Postgres | 200x slower |
| Full-Text Search | Elasticsearch | Postgres LIKE | 600x slower |
| Stream Processing | Kafka + Flink | Batch ETL | 3000x slower |
| Graph Traversal | Neo4j | SQL with CTEs | 2000x slower |

**Shopify's Architecture (handles Black Friday traffic):**
- **Checkout Events**: Apache Kafka (100 brokers, 7-day retention)
- **Product Catalog**: MySQL with Vitess sharding (50 shards)
- **Analytics Rollups**: ClickHouse (20 nodes)
- **ML Training Data**: Parquet on S3
- **User Sessions**: Redis Cluster (50 nodes)

Total monthly cost with polyglot approach: $240K
If using only MySQL: $2.4M/month (10x more expensive)

## 3.5 Format Selection: A Decision Framework

After all the theory, here's what actually matters when choosing formats:

### The Quick Decision Matrix

| Use Case | Format | Why | Watch Out For |
|----------|--------|-----|---------------|
| Config files | JSON/YAML | Human-readable | Humans will mess it up |
| API interchange | JSON | Everything speaks it | Type ambiguity |
| Analytics/ML | Parquet | Fast, compressed, typed | Not human-readable |
| Streaming | Avro | Schema evolution | Learning curve |
| In-memory | Arrow | Zero-copy | Still maturing |
| Legacy systems | CSV | It just works | It """works""" |
| Please no | XML | Job security | It's not 2001 |

### The Real Questions to Ask

**1. Who needs to read this data?**
- Humans debugging → JSON, CSV
- Other systems → Parquet, Avro, Arrow
- Both → You need two formats (seriously)

**2. How big is the data?**
- Under 100MB → Whatever's convenient
- 100MB - 10GB → Parquet with compression
- Over 10GB → Parquet with partitioning
- Over 1TB → Parquet with partitioning AND a data catalog

**3. How often does the schema change?**
- Never → Parquet, strict validation
- Occasionally → Avro (built for schema evolution)
- Constantly → JSON blob with validation layer
- Every sprint → You have bigger problems

**4. What's the access pattern?**
- Full scans → Parquet (columnar wins)
- Point lookups → Avro or key-value store
- Time-range queries → Parquet sorted by timestamp
- Real-time → Arrow or streaming format

### The Migration Reality

Format migrations are expensive. A company I worked with spent 18 months migrating from JSON to Parquet. They estimated 3 months initially. The actual breakdown:

- Month 1-3: Converting the easy stuff (30% of data)
- Month 4-6: Discovering edge cases in "clean" data
- Month 7-12: Rewriting consumers that assumed JSON quirks
- Month 13-15: Dealing with the data that "can't" be migrated
- Month 16-18: Cleaning up the dual-format mess

**The lesson:** Choose your format carefully upfront. Migration costs are always 3-6x what you estimate.

## Quick Wins Box: Format Fixes

**1. Validate JSON before parsing:**
```python
import json

def safe_json_load(text):
    try:
        return json.loads(text), None
    except json.JSONDecodeError as e:
        return None, f"Invalid JSON at position {e.pos}: {e.msg}"
```

**2. Convert CSV to Parquet for any analysis work:**
```python
import pandas as pd

# One-time conversion that pays dividends forever
df = pd.read_csv('data.csv')
df.to_parquet('data.parquet', compression='zstd')
# Delete the CSV if you dare
```

**3. Add format detection to your ingestion:**
```python
def detect_and_load(file_path):
    """Load any supported format automatically."""
    with open(file_path, 'rb') as f:
        header = f.read(8)
    
    if header.startswith(b'PAR1') or header.endswith(b'PAR1'):
        return pd.read_parquet(file_path)
    elif header.startswith(b'PK'):
        return pd.read_excel(file_path)
    elif header[0:1] == b'{' or header[0:1] == b'[':
        return pd.read_json(file_path)
    else:
        return pd.read_csv(file_path)  # CSV is the fallback
```

## Your Homework

### Exercise 1: The Format Audit (30 minutes)

List every file format in your data pipeline. For each one, answer:
1. Why was this format chosen?
2. Is that reason still valid?
3. What would it cost to change it?

I bet at least one format is "because that's how we've always done it."

### Exercise 2: The Compression Test (15 minutes)

Take your largest dataset and try different formats:
```python
import pandas as pd
import os

df = pd.read_csv('your_data.csv')

# Test different formats
df.to_csv('test.csv', index=False)
df.to_parquet('test_snappy.parquet', compression='snappy')
df.to_parquet('test_zstd.parquet', compression='zstd')
df.to_parquet('test_gzip.parquet', compression='gzip')

for f in ['test.csv', 'test_snappy.parquet', 'test_zstd.parquet', 'test_gzip.parquet']:
    print(f"{f}: {os.path.getsize(f) / 1024 / 1024:.1f} MB")
```

### Exercise 3: The JSON Horror Hunt (20 minutes)

Find a JSON API you use regularly. Look for:
- Numbers stored as strings
- Dates in non-ISO format
- Nested redundant wrappers
- Boolean-like strings ("yes", "no", "true", "false")

Document at least three issues. Consider sending the API maintainers a polite note.

## Parting Thoughts

File formats are like plumbing - invisible when they work, catastrophic when they don't. JSON will keep lying to you, CSV will keep losing your types, and Parquet will keep saving your analytics team's sanity.

The format wars aren't going away. New formats will emerge, old formats will persist long past their expiration date, and you'll spend more time than you'd like converting between them. The best you can do is understand the trade-offs and choose deliberately.

In Chapter 4, we'll finally talk about money - the hidden costs of bad data decisions, the ROI of getting this right, and how to make the business case for all the infrastructure work we've been discussing.

Until then, go check how your data is actually stored. I promise you'll find at least one format decision that makes you question your predecessors' sanity.

---

*P.S. - Somewhere in your stack, there's a JSON file with a date formatted as "01/02/03". Nobody knows if that's January 2nd 2003, February 1st 2003, or February 3rd 2001. It's been that way for years. Everyone's afraid to touch it.*

---

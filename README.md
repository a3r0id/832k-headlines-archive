# 832k Headlines Archive

A SQLite archive of **832,207** news headlines and related records collected roughly from **July 2022 through April 2026**, released under the [MIT License](LICENSE).

The dataset lives in a single file:

| File | Size | Format |
|------|------|--------|
| [`database.db`](database.db) | ~789 MB | SQLite 3 (UTF-8) |

> **Clone note:** this file is larger than GitHub’s normal 100 MB blob limit. Install [Git LFS](https://git-lfs.com) before cloning or pushing (`git lfs install`).

---

## What’s in the box

One table, `headlines`, with no secondary indexes:

```sql
CREATE TABLE headlines (
    id        INTEGER PRIMARY KEY,
    dataNode  INTEGER,  -- collection stream (see below)
    hash      TEXT,     -- SHA-1 of `text`
    text      TEXT,     -- headline / trend string
    link      TEXT,     -- article or search URL
    pubDate   TEXT,     -- publication time (UTC string)
    source    TEXT,     -- outlet or feed name
    urls      TEXT,     -- JSON array of related media URLs
    content   TEXT,     -- longer body / tweet-style text (often null)
    kwargs    TEXT,     -- JSON object of extra metadata
    sentiment TEXT,     -- JSON sentiment analysis payload
    timenow   TEXT      -- unix timestamp when the row was ingested
);
```

### Field reference

| Column | Type | Notes |
|--------|------|-------|
| `id` | integer | Stable primary key (1 … 832207). |
| `dataNode` | integer | Which collection stream produced the row (`0`–`3`). |
| `hash` | text | `sha1(text)` hex digest. Nearly unique (~203 duplicate hashes). |
| `text` | text | Headline or trending topic. Avg length ~91 chars. |
| `link` | text | Destination URL (short links common: `reut.rs`, `fxn.ws`, `nyti.ms`, …). |
| `pubDate` | text | UTC timestamp string — formats vary by stream (see below). |
| `source` | text | Outlet name. **3,222** distinct values; 18 rows have `NULL`. |
| `urls` | text | JSON array of image/media URLs, e.g. `["https://…"]`. Empty `[]` on ~140k rows. |
| `content` | text | Optional longer text. **Null on ~280k rows** (especially `dataNode` 1 and 3). |
| `kwargs` | text | JSON object. Empty `{}` for most rows; Twitter trends store volume/query here. |
| `sentiment` | text | JSON with label + score (see below). Present on every row. |
| `timenow` | text | Ingestion time as a unix float string (collection window: 2022-07-25 → 2026-04-14 UTC). |

### `dataNode` streams

Rows come from four collection streams:

| `dataNode` | Rows | What it looks like |
|------------|------|--------------------|
| `0` | 401,869 | Short news headlines with social-style `content` always present. Top sources: Reuters, Fox News, NYT, CBS, AP, … |
| `1` | 82,534 | **Twitter trending topics** only (`source = 'Twitter'`). `content` is always null; `kwargs` holds `tweet_volume` + `query`. |
| `2` | 178,627 | Full news-article headlines (often includes outlet name in `text`). Mixed `content` coverage. |
| `3` | 169,177 | News headlines with **no** `content` body. Links skew toward Yahoo / aggregator URLs. |

If you only want classic news headlines (not Twitter trends):

```sql
SELECT * FROM headlines WHERE dataNode != 1;
```

---

## Coverage snapshot

**By publication year** (`pubDate` prefix):

| Year | Rows |
|------|------|
| 2022 | 285,144 |
| 2023 | 325,594 |
| 2024 | 108,500 |
| 2025 | 101,771 |
| 2026 | 11,191 |
| other / junk | 7 |

Dense collection runs mid-2022 through mid-2023 (~50k/month), then settles around ~7–10k/month. A handful of rows have outlier `pubDate` values (one `1970-01-01` tombstone, a few pre-2022 stamps).

**Top sources**

| Source | Rows |
|--------|------|
| Reuters | 109,762 |
| Twitter | 82,534 |
| Fox News | 62,418 |
| Associated Press | 52,173 |
| CBS News | 32,991 |
| New York Times | 29,986 |
| Daily Caller | 27,364 |
| Al Jazeera English | 24,757 |
| HuffPost | 23,998 |
| NBC News | 22,387 |

**Sentiment** (`json_extract(sentiment, '$.overall_sentiment')`):

| Label | Rows |
|-------|------|
| Neutral | 314,750 |
| Negative | 301,329 |
| Positive | 216,128 |

Scores range roughly **-0.99 … +0.99** (mean ≈ **-0.065**).

---

## `pubDate` formats

Timestamps are UTC strings, but the shape differs by stream:

```text
2022-07-25 17:43:06 UTC              -- dataNode 0 (space-separated)
2022-07-25 14:02:06.534521 UTC       -- dataNode 1 (fractional seconds)
2024-05-01T12:34:56Z UTC             -- dataNode 2/3 (ISO-8601 with Z)
```

For range filters, prefix match on the date portion usually works:

```sql
SELECT * FROM headlines
WHERE pubDate LIKE '2024-06%'
LIMIT 20;
```

Or normalize in Python with `dateutil.parser` / a small custom parser.

---

## Sentiment JSON shape

```json
{
  "sentence": "…headline text…",
  "overall_sentiment": "Positive",
  "overall_sentiment_score": 0.1531,
  "scores": [
    { "positive": 0.132, "negative": 0.113, "neutral": 0.755 }
  ]
}
```

Query labels/scores directly with SQLite JSON functions:

```sql
SELECT
  json_extract(sentiment, '$.overall_sentiment') AS label,
  json_extract(sentiment, '$.overall_sentiment_score') AS score,
  COUNT(*) AS n
FROM headlines
GROUP BY 1
ORDER BY n DESC;
```

---

## Twitter trends (`kwargs`)

Only `dataNode = 1` rows populate `kwargs` meaningfully:

```json
{ "tweet_volume": 41407, "query": "%23BBCOurNextPM" }
```

```sql
SELECT
  text,
  json_extract(kwargs, '$.tweet_volume') AS volume,
  link
FROM headlines
WHERE dataNode = 1
ORDER BY volume DESC
LIMIT 20;
```

---

## Quick start (Python)

```python
import json
import sqlite3

conn = sqlite3.connect("database.db")
conn.row_factory = sqlite3.Row

# Latest Reuters headlines
rows = conn.execute(
    """
    SELECT id, text, link, pubDate, source,
           json_extract(sentiment, '$.overall_sentiment') AS label
    FROM headlines
    WHERE source = 'Reuters'
      AND dataNode != 1
    ORDER BY pubDate DESC
    LIMIT 10
    """
).fetchall()

for row in rows:
    print(dict(row))

# Parse media URLs
row = conn.execute(
    "SELECT text, urls FROM headlines WHERE urls != '[]' LIMIT 1"
).fetchone()
print(row["text"], json.loads(row["urls"]))

conn.close()
```

### Useful one-liners

```sql
-- Row count
SELECT COUNT(*) FROM headlines;

-- Distinct outlets
SELECT COUNT(DISTINCT source) FROM headlines;

-- Headlines mentioning a keyword
SELECT pubDate, source, text
FROM headlines
WHERE text LIKE '%inflation%'
ORDER BY pubDate DESC
LIMIT 50;

-- Daily volume
SELECT substr(pubDate, 1, 10) AS day, COUNT(*) AS n
FROM headlines
WHERE pubDate LIKE '2023-%'
GROUP BY 1
ORDER BY 1;
```

### Performance tip

There are **no indexes** besides the primary key. Filters on `source`, `pubDate`, or `dataNode` scan the full table (seconds on a laptop). If you query repeatedly, create your own indexes:

```sql
CREATE INDEX idx_headlines_source ON headlines(source);
CREATE INDEX idx_headlines_pubdate ON headlines(pubDate);
CREATE INDEX idx_headlines_datanode ON headlines(dataNode);
```

---

## Data quality notes

- **Near-duplicate headlines** exist across outlets and streams; `hash` is based only on `text`, so identical wording collapses, but rewrites do not.
- **`content` is missing** for all of `dataNode` 1 and 3, and for ~16% of `dataNode` 2.
- **Sentiment labels** are machine-generated and can look odd on short/fragmented headlines (e.g. crime stories labeled “Positive”).
- A few rows are placeholders (`source = '[Removed]'`, epoch `pubDate`).
- Links include shortened URLs and may have expired since collection.

---

## License

MIT — see [LICENSE](LICENSE). Downstream users may use this archive for research, NLP training, journalism tooling, etc. Article text and linked media remain subject to their original publishers’ rights; this release covers the collected metadata archive as distributed here.

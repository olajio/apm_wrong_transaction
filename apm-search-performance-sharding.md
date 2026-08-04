# APM Search Performance: Sharding Strategy Analysis

## Current State

- **APM indices:** 1 primary shard, 1 replica shard per backing index
- **Cluster version:** Elastic Stack 9.3
- **Reported symptom:** Slow APM UI performance generally (transaction list, trace waterfall, service map, metrics)

---

## Why Sharding Directly Affects APM Search Performance

Every APM UI action — opening a transaction list, loading a trace sample, rendering the service map — fires one or more Elasticsearch queries that fan out to **every shard** in the targeted data streams. The number, size, and health of those shards determines how fast those queries return.

| Sharding problem | Search impact |
|---|---|
| Shards too large (> 50 GB) | Slow query execution; long recovery time if a node fails |
| Shards too small (< 1 GB) | Excessive coordination overhead; master node heap pressure; queries hit many tiny segments |
| Too many segments per shard | Merge overhead slows both indexing and search; high segment count = more heap per shard |
| Single primary shard | Writes bottleneck on one node; no ingest parallelism; shard grows unboundedly until rollover |
| Insufficient replicas | All search load concentrates on primary; no read parallelism across copies |

---

## Step 1 — Diagnose Before Changing Anything

Do not adjust sharding until you know your actual shard sizes. Guessing leads to over-sharding, which is harder to fix than under-sharding.

### 1a. Check current shard sizes across all APM data streams

```
GET _cat/shards/traces-apm*,metrics-apm*,logs-apm*?v&h=index,shard,prirep,state,docs,store&s=store:desc
```

This returns each backing index's shards sorted by size (largest first). Look for:
- Shards well above 50 GB → under-sharded, need more primaries or more frequent rollover
- Shards well below 1 GB → over-sharded or rolling over too frequently
- The 10–40 GB range is the target

### 1b. Check segment counts

```
GET _cat/segments/traces-apm*?v&h=index,shard,prirep,segment,size,size.memory&s=index
```

High segment counts (dozens or hundreds per shard on a "hot" index) indicate that force-merge has not run or that refresh intervals are too short.

### 1c. Check ILM rollover policy

```
GET _ilm/policy/traces-apm.policy
GET _ilm/policy/metrics-apm.policy
GET _ilm/policy/logs-apm.policy
```

Check whether rollover is triggered by `max_age`, `max_size`, or `max_primary_shard_size`. The last one is the recommended trigger (see ILM section below).

### 1d. Check current ingest rate per data stream

```
GET _cat/indices/traces-apm*?v&h=index,docs.count,store.size,creation.date.string&s=creation.date.string:desc
```

This tells you how fast each backing index is growing. Divide the size by the number of days since creation to estimate GB/day — the key input to your shard-count calculation.

---

## Step 2 — Understand the Correct Shard Sizing Model

### Official guidance (Elastic 8.3+ / 9.x)

Elastic's current guidance ([Size your shards](https://www.elastic.co/docs/deploy-manage/production-guidance/optimize-performance/size-shards)):

> Keep shard sizes in the **10–50 GB range**. The target sweet spot for search-heavy workloads is **20–40 GB**.

> The "20 shards per GB of heap" rule **was deprecated in Elasticsearch 8.3** and should not be followed. Do not use it for sizing decisions in 9.x.

> Keep the per-shard document count **below 200 million** documents.

> Each non-frozen data node supports up to **1,000 shards**. Exceeding this degrades cluster stability.

### Why 1 primary shard may not be enough

With 1 primary shard:
- All writes go to a single node — ingest throughput is bounded by that one node's CPU and I/O
- As the backing index grows between rollovers, the single shard can exceed the 50 GB threshold
- After rollover the old shard is read-only and searchable, but the hot shard remains a single point of write contention

For **high-throughput APM environments** (hundreds of services, thousands of transactions per second), 2–4 primary shards on the `traces-apm` data stream is common.

### Why 1 replica is a reasonable starting point but may limit search throughput

1 replica gives you:
- **High availability:** if the primary node fails, the replica is promoted
- **2x read parallelism:** queries can be served by either the primary or the replica

If APM search is still slow after fixing shard sizes, increasing replicas to 2 (giving 3 total copies) allows more concurrent searches without additional primary shards. Each additional replica adds a full copy of all data, so weigh storage cost accordingly.

---

## Step 3 — Calculate the Right Number of Primary Shards

Use this formula once you know your daily ingest rate:

```
number_of_primary_shards = ceil(daily_ingest_GB × rollover_days / target_shard_size_GB)
```

Example:
- `traces-apm` grows at 30 GB/day
- ILM rolls over every 7 days (or at 40 GB, whichever comes first)
- Target shard size: 30 GB
- `ceil(30 × 7 / 30)` = `ceil(7)` = **7 primary shards** (adjust down if you roll over on size before age)

If you roll over on `max_primary_shard_size: 40gb` (recommended), the calculation simplifies: with 1 primary shard, rollover fires automatically when that shard hits 40 GB, so each backing index stays near 40 GB without manual calculation. You only need more than 1 primary shard if your ingest rate is so high that a single shard cannot absorb writes fast enough.

---

## Step 4 — Fix the ILM Rollover Trigger

This is the **highest-impact, lowest-risk change** and should be done before adjusting shard counts.

### The problem with time-based rollover only

If the current policy rolls over on `max_age: 1d` (daily), you can end up with:
- Very small indices during low-traffic periods (nights, weekends)
- Very large indices during high-traffic periods
- A cluster full of tiny shards that consume heap and coordination overhead but store almost no data

### Recommended: size-based rollover

```json
PUT _ilm/policy/traces-apm.policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_primary_shard_size": "40gb",
            "max_age": "30d"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "2d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "7d",
        "actions": {
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

Key points:
- `max_primary_shard_size: 40gb` triggers rollover when the primary shard reaches 40 GB — shards never balloon beyond the target size
- `max_age: 30d` is a safety net so very-low-traffic indices still roll over eventually (prevents a single index from living forever)
- The `warm` phase shrinks to 1 shard and force-merges to 1 segment, dramatically reducing search overhead on historical data
- Adjust `min_age` values to match your retention requirements

> Apply equivalent policies to `metrics-apm.policy` and `logs-apm.policy`.

---

## Step 5 — Customize the Index Template for Primary Shard Count

If your ingest rate is high enough that a single primary shard is a write bottleneck, increase the number of primaries via the `traces-apm@custom` component template. **Do not modify the managed `traces-apm` template directly** — Elastic overwrites it on upgrade.

```json
PUT _component_template/traces-apm@custom
{
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1
    }
  }
}
```

This takes effect on the **next rollover** (new backing index). Existing backing indices are unaffected. Trigger a manual rollover to apply immediately:

```
POST traces-apm-default/_rollover
```

Repeat with `metrics-apm@custom` and `logs-apm@custom` if those data streams also show large shards or write bottlenecks. Note that `metrics-apm` and `logs-apm` typically ingest at lower rates than `traces-apm` and may be fine with 1 primary shard.

> If you applied the `traces-apm@custom` workaround for the `timestamp.us` known bug (see `investigation.md`), **merge both changes into a single template** rather than overwriting one with the other:

```json
PUT _component_template/traces-apm@custom
{
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "timestamp": {
          "properties": {
            "us": { "type": "long" }
          }
        }
      }
    }
  }
}
```

---

## Step 6 — Force-Merge Historical (Read-Only) Indices

Once an index rolls over to warm/cold, it is read-only. APM queries that scan historical data still hit these indices. A high segment count on old indices significantly slows searches. Force-merging to 1 segment per shard makes searches as fast as possible:

```
POST traces-apm-default-<old-index-name>/_forcemerge?max_num_segments=1
```

This is CPU- and I/O-intensive. Run it during off-peak hours, or let the ILM `warm` phase handle it automatically via the `forcemerge` action (as shown in the ILM policy above).

To find indices that have not been force-merged:

```
GET _cat/segments/traces-apm*?v&h=index,shard,segment,size&s=index
```

Any read-only index with more than 1–2 segments per shard is a candidate.

---

## Step 7 — Hot/Warm/Cold Node Roles (if not already configured)

APM data has a natural time-decay access pattern: recent data is queried constantly, data older than a few days is queried rarely. Hot/warm/cold tiering maps that access pattern to hardware:

| Tier | Node type | Hardware | Data age | ILM phase |
|---|---|---|---|---|
| Hot | `data_hot` | NVMe SSD, high CPU | 0–2 days | `hot` |
| Warm | `data_warm` | SSD or spinning disk | 2–7 days | `warm` |
| Cold | `data_cold` | Spinning disk or object storage | 7–30 days | `cold` |
| Frozen | `data_frozen` | Object storage (S3/GCS/Azure) | > 30 days | `frozen` |

Even a two-tier setup (hot + warm) significantly reduces search latency for current data because the hot nodes handle the write-heavy current index without competing with searches over large historical indices.

To assign a node to the hot tier, set in `elasticsearch.yml`:

```yaml
node.roles: [data_hot, ingest]
```

The ILM policy's `warm` phase `migrate` action moves data automatically.

---

## Step 8 — Tune Refresh Interval Under Heavy Ingest

By default, Elasticsearch refreshes every 1 second, making documents searchable. Each refresh creates a new segment. Under heavy APM ingest, this produces many tiny segments that slow search and increase merge pressure.

For the hot backing index under heavy ingest, increasing the refresh interval reduces segment churn:

```json
PUT traces-apm-default/_settings
{
  "refresh_interval": "10s"
}
```

Or set it in the index template so all new backing indices inherit it:

```json
PUT _component_template/traces-apm@custom
{
  "template": {
    "settings": {
      "refresh_interval": "10s",
      "number_of_shards": 2,
      "number_of_replicas": 1
    }
  }
}
```

Trade-off: APM data will appear in the UI with up to 10 seconds delay. For near-real-time monitoring this is usually acceptable; for live alerting on streaming events it may not be.

---

## Summary: Recommended Action Plan

| Priority | Action | Effort | Expected impact |
|---|---|---|---|
| 1 | Run diagnostic queries (shard sizes, segment counts, ILM policies) | 15 min | Baseline understanding |
| 2 | Switch ILM rollover trigger from `max_age` to `max_primary_shard_size: 40gb` | 30 min | Prevents future oversized shards; eliminates tiny empty shards |
| 3 | Add `forcemerge` and `shrink` to the ILM `warm` phase | 30 min | Significant search speedup on historical data |
| 4 | If current hot shard > 50 GB: increase `number_of_shards` in `traces-apm@custom` | 30 min | Reduces write bottleneck, shrinks individual shard size |
| 5 | If search is still slow: increase `number_of_replicas` to 2 | 15 min | More read parallelism; increases storage cost |
| 6 | Add hot/warm/cold node tiers | Days–weeks | Largest overall throughput and cost improvement; requires infrastructure change |
| 7 | Tune `refresh_interval` to 10s on hot indices | 10 min | Reduces segment churn; small latency trade-off |

---

## APM Data Streams Quick Reference

| Data stream | Content | Typical size rank | Notes |
|---|---|---|---|
| `traces-apm-*` | Transactions + spans | Largest | Most impacted by high `transaction_max_spans` |
| `metrics-apm.*-*` | APM agent metrics | Medium | Consider TSDS for further storage reduction |
| `logs-apm.*-*` | APM errors and logs | Smallest | Often rolled over infrequently; check for large shards |

---

## References

| Resource | Link |
|---|---|
| Size your shards (official Elastic guidance) | [elastic.co · size-shards](https://www.elastic.co/docs/deploy-manage/production-guidance/optimize-performance/size-shards) |
| Elasticsearch shard & node sizing best practices | [elastic.co · search-labs/elasticsearch-node-shard-size-best-practices](https://www.elastic.co/search-labs/blog/elasticsearch-node-shard-size-best-practices) |
| How many shards should I have? | [elastic.co · blog/how-many-shards](https://www.elastic.co/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster) |
| APM data streams | [elastic.co · apm/data-streams](https://www.elastic.co/docs/solutions/observability/apm/data-streams) |
| APM ILM | [elastic.co · apm/index-lifecycle-management](https://www.elastic.co/docs/solutions/observability/apm/index-lifecycle-management) |
| APM processing and performance | [elastic.co · apm-processing-and-performance](https://www.elastic.co/guide/en/observability/current/apm-processing-and-performance.html) |
| Benchmarking Elasticsearch for logs and metrics | [elastic.co · blog/benchmarking-and-sizing](https://www.elastic.co/blog/benchmarking-and-sizing-your-elasticsearch-cluster-for-logs-and-metrics) |
| Kibana issue #161984 — APM metrics shard breakdown | [github.com · kibana/issues/161984](https://github.com/elastic/kibana/issues/161984) |

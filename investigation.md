# APM: Wrong Transaction Details Displayed in Kibana

## The Issue

In the Kibana Observability APM app, opening `CAVIAR Haystack API → Transactions → GET EquityVol/GetEquityDividend` and picking a trace sample renders a waterfall whose root/entry transaction is `caviar-calculate-risk` — not the selected `GET EquityVol/GetEquityDividend`. A banner on the waterfall reports:

> ⚠ The number of items in this trace is **135,224** which is higher than the current limit of **2,000**. Please increase the limit via `xpack.apm.ui.maxTraceItems` to see the full trace.

The panel also shows an **"Incomplete trace"** tag.

---

## Root Cause

Two things are happening at once.

### 1. Expected behavior — the trace-samples panel shows the whole distributed trace

The Trace samples section on the Transaction detail page renders the **entire distributed trace** that the selected transaction participated in, rooted at that trace's first transaction. Because `GET EquityVol/GetEquityDividend` is a downstream call executed inside a larger `caviar-calculate-risk` request, the waterfall correctly opens at `caviar-calculate-risk`. The selected transaction is meant to appear further down the waterfall as one of its children.

### 2. The trace is truncated by `xpack.apm.ui.maxTraceItems`

Kibana's APM UI caps the number of items (transactions + spans) it fetches per trace at `xpack.apm.ui.maxTraceItems` (**default: 5,000; this cluster is set to 2,000 — below the default**). When a trace has 135,224 items, the query returns only the **earliest 2,000 by timestamp** — the root transaction and its immediate children — and drops everything else.

The selected `GET EquityVol/GetEquityDividend` transaction starts later in the trace chronologically, so it falls outside the 2,000-item window and **never renders in the waterfall**. That is why the details on screen appear to belong to a different transaction.

**The 135,224 vs 2,000 warning is directly causing the observed behavior.**

---

## `xpack.apm.ui.maxTraceItems` — Complete Reference

### Default value

**5,000** — confirmed from [Kibana source (`x-pack/solutions/observability/plugins/apm/server/index.ts`)](https://github.com/elastic/kibana/blob/main/x-pack/solutions/observability/plugins/apm/server/index.ts):

```ts
maxTraceItems: schema.number({ defaultValue: 5000 })
```

This cluster is set to **2,000**, which is below the default. This is making the problem worse than the out-of-box behaviour.

### Acceptable values

Any positive integer — the schema imposes no enforced minimum or maximum. However there is a **practical hard ceiling**:

**Maximum safe value: 10,000**

This is imposed by Elasticsearch's default `index.max_result_window` of 10,000, which caps the `size` parameter on any search query. Setting `maxTraceItems` above 10,000 will cause Elasticsearch to reject the query with a `Result window is too large` error unless `index.max_result_window` has been explicitly raised on the APM indices. Batched ES pagination for this query was tracked as a future improvement in [Kibana issue #164937](https://github.com/elastic/kibana/issues/164937) but was not the default behaviour in most 8.x releases.

**Practical guidance:**

| Value | Notes |
|---|---|
| 2,000 | Your current value — below default, worsens the problem |
| 5,000 | Default — safe starting point |
| 10,000 | Maximum safe value without raising `index.max_result_window` |
| > 10,000 | Requires raised `index.max_result_window` + Kibana version with batched ES pagination |

### Why is the behaviour inconsistent? (works sometimes, fails other times)

There is **no ratio** governing this — it is a binary outcome based on where the selected transaction lands chronologically inside the trace.

When a trace is fetched, Kibana queries Elasticsearch for up to `maxTraceItems` transactions and spans, **sorted by timestamp ascending (start time)**. It takes the first N items that fit. Errors are fetched separately and do not count against the limit ([PR #56111](https://github.com/elastic/kibana/pull/56111)).

For `GET EquityVol/GetEquityDividend`:

| Trace sample selected | Outcome |
|---|---|
| Small, independent trace (< 2,000 items) | ✅ All items fit — waterfall renders correctly |
| The 135,224-item `caviar-calculate-risk` trace (your transaction starts late) | ❌ Transaction falls outside the first 2,000 chronological items — wrong/missing waterfall |

The APM UI rotates through trace samples when you navigate the latency distribution. When it picks a sample from a small trace → looks fine. When it picks the massive trace → wrong details. **This is the inconsistency you observed** — it tracks with which sample the UI is presenting, not random noise.

Additionally, as noted in [issue #120464](https://github.com/elastic/kibana/issues/120464), if a **parent span is truncated**, its entire subtree also disappears from the waterfall, compounding the problem.

### Impact of setting `maxTraceItems` too high

| Risk | Detail |
|---|---|
| **Elasticsearch query rejection** | Above 10,000 ES rejects the query unless `index.max_result_window` is raised |
| **Browser memory / freeze** | The waterfall renders as a DOM tree; 10,000 nodes is already heavy |
| **Kibana server memory** | Full payload loaded into Kibana's Node.js process before sending to the browser |
| **Slow UI response** | Large ES queries compete with other Kibana queries under heavy APM ingest |
| **No upside above trace size** | Setting 50,000 gives zero benefit over 10,000 if ES caps it at 10,000 anyway |

---

## Remediation — Ordered by Lowest Effort First

### Step 1 — Immediate workaround (no config change required)

- Click **"View full trace"** or **"Open in Discover"** on the trace sample panel. This takes you directly to the Elasticsearch document for the specific transaction you selected, bypassing the waterfall entirely. You will see the correct transaction details.
- Alternatively, pick a different trace sample from the latency distribution — one from a small trace (< 2,000 items) that does not involve `caviar-calculate-risk` as the root.

### Step 2 — Restore the default (Kibana restart only, ~5 min)

Your current value of 2,000 is below the shipped default of 5,000. Raise it to the default first:

```yaml
# kibana.yml (self-managed)
# or Kibana user settings on Elastic Cloud
xpack.apm.ui.maxTraceItems: 5000
```

Restart Kibana. This resolves the issue for any trace smaller than 5,000 items and correctly benchmarks the problem against the designed default.

### Step 3 — Raise to the safe ceiling (Kibana restart only, ~5 min)

If the problem persists for the specific 135k-item trace:

```yaml
xpack.apm.ui.maxTraceItems: 10000
```

Do **not** go above 10,000 without first verifying your Kibana version supports batched ES pagination for this query ([issue #164937](https://github.com/elastic/kibana/issues/164937)).

### Step 4 — Check APM agent `transaction_max_spans` (config change, no code change)

Before touching any application code, check the APM agent configuration on the `caviar-calculate-risk` service. The APM agent has a `transaction_max_spans` setting (default: **500 per transaction**). If this was raised, lower it back:

```yaml
# Java agent
transaction_max_spans: 500
```

Relevant agent docs:
- [Java agent — transaction_max_spans](https://www.elastic.co/guide/en/apm/agent/java/current/config-core.html#config-transaction-max-spans)
- [Python agent — transaction_max_spans](https://www.elastic.co/guide/en/apm/agent/python/current/configuration.html#config-transaction-max-spans)
- [Node.js agent — transactionMaxSpans](https://www.elastic.co/guide/en/apm/agent/nodejs/current/configuration.html#transaction-max-spans)

### Step 5 — Fix the instrumentation (code change)

A single trace with 135,224 items is almost always an instrumentation defect. Investigate `caviar-calculate-risk` for:

- **Per-iteration spans in a loop** — per-row DB calls, per-item HTTP calls, per-message queue publishes. Batch the operation or wrap the loop in a single parent span.
- **Trace-context leakage across requests** — background workers, async jobs, or message consumers that continue an incoming trace context instead of starting a new transaction. Independent requests get stitched into one giant trace.
- **`transaction_max_spans` raised too high in the APM agent** — see Step 4.
- **Recursive fan-out** — `caviar-calculate-risk` calling itself or fanning out to thousands of downstream calls. Consider downstream sampling or splitting into separate traces.

**Recommended starting point:** From Service inventory, open `caviar-calculate-risk`, pick one of its transactions, and inspect the span breakdown by type (db, http, external, custom) to identify which span category is producing tens of thousands of items.

---

## References

| Resource | Link |
|---|---|
| APM Settings in Kibana (official docs) | [elastic.co/guide/en/kibana/8.18/apm-settings-kb.html](https://www.elastic.co/guide/en/kibana/8.18/apm-settings-kb.html) |
| Kibana source — maxTraceItems defaultValue: 5000 | [github.com/elastic/kibana/.../apm/server/index.ts](https://github.com/elastic/kibana/blob/main/x-pack/solutions/observability/plugins/apm/server/index.ts) |
| PR #56111 — Fixes maxTraceItems in waterfall & error queries | [github.com/elastic/kibana/pull/56111](https://github.com/elastic/kibana/pull/56111) |
| Issue #164937 — Trace waterfall pagination for big traces | [github.com/elastic/kibana/issues/164937](https://github.com/elastic/kibana/issues/164937) |
| Issue #118282 — Add more info to the "exceeds max" callout | [github.com/elastic/kibana/issues/118282](https://github.com/elastic/kibana/issues/118282) |
| Issue #118551 — Link to maxTraceItems config in trace callout | [github.com/elastic/kibana/issues/118551](https://github.com/elastic/kibana/issues/118551) |
| Issue #120464 — Missing trace items shouldn't break waterfall | [github.com/elastic/kibana/issues/120464](https://github.com/elastic/kibana/issues/120464) |

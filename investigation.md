# APM: Wrong Transaction Details Displayed in Kibana

## The Issue

In the Kibana Observability APM app, opening `CAVIAR Haystack API → Transactions → GET EquityVol/GetEquityDividend` and picking a trace sample renders a waterfall whose root/entry transaction is `caviar-calculate-risk` — a transaction from a **different service** — not the selected `GET EquityVol/GetEquityDividend`. A banner on the waterfall reports:

> ⚠ The number of items in this trace is **135,224** which is higher than the current limit of **2,000**. Please increase the limit via `xpack.apm.ui.maxTraceItems` to see the full trace.

The panel also shows an **"Incomplete trace"** tag.

---

## ⚠ Version 9.3 Known Bug — Check This First

The cluster is on Elastic Stack **9.3**. There is a **confirmed known bug** in versions **≥ 9.3.0 and < 9.3.2** that directly affects the trace waterfall:

**Symptom:** Spans in the trace waterfall render aligned on the left instead of at their actual start time. The waterfall timeline is visually wrong — spans appear to all start at time zero rather than at their true offset within the trace.

**Cause:** The field `timestamp.us` is not explicitly mapped as `long` in the APM index templates from Elasticsearch 8.15 onwards. When the APM Server uses a non-Elasticsearch output (Logstash, Redis, Kafka, Console) to ship data to Elasticsearch, the agent can serialize `timestamp.us` as a float when the value exceeds `Number.MAX_SAFE_INTEGER`. Elasticsearch then infers the field type as `float`, breaking the waterfall visualization.

**Fix released in:** 8.19.13, 9.2.7, **9.3.2**, 9.4.0 — tracked in [elasticsearch#143173](https://github.com/elastic/elasticsearch/pull/143173)

**First action — detect whether you are affected:**

```
GET traces-apm-default/_mapping/field/timestamp.us
```

If the result shows `type: float` instead of `type: long`, you are affected.

**If you cannot upgrade to 9.3.2 immediately — workaround:**

Add an explicit `long` mapping to the `traces-apm@custom` component template:

```json
PUT _component_template/traces-apm@custom
{
  "template": {
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

Then trigger a manual rollover on the affected data streams so the new mapping takes effect on the next backing index:

```
POST traces-apm-default/_rollover
```

**After upgrading to 9.3.2:** Remove the workaround template to avoid conflicts:

```
DELETE _component_template/traces-apm@custom
POST traces-apm-default/_rollover
```

> This bug may be contributing to the wrong-details symptom independently of the `maxTraceItems` truncation. Resolve the version issue before drawing further conclusions from the waterfall display.

---

## Root Cause

Three factors are compounding. Each one on its own might produce incorrect-looking waterfall output; together they make the problem harder to diagnose.

### Factor 1 — Expected behavior: the trace-samples panel shows the whole distributed trace

The Trace samples section on the Transaction detail page renders the **entire distributed trace** that the selected transaction participated in, rooted at the trace's first transaction. The waterfall always opens at the root service, not at the service you navigated from. This means `caviar-calculate-risk` appearing at the top is, in isolation, not necessarily wrong — it depends on whether it is the legitimate initiator of the trace (see Factor 3).

### Factor 2 — Trace truncation by `xpack.apm.ui.maxTraceItems`

Kibana's APM UI caps the number of items (transactions + spans) it fetches per trace at `xpack.apm.ui.maxTraceItems` (**default: 5,000; this cluster is set to 2,000 — below the default**). When a trace has 135,224 items, the query returns only the **earliest 2,000 by timestamp** — the root transaction and its immediate children — and drops everything else.

The selected `GET EquityVol/GetEquityDividend` transaction starts later in the trace chronologically, so it falls outside the 2,000-item window and **never renders in the waterfall**. That is why the panel shows wrong details and the "Incomplete trace" tag.

### Factor 3 — Uncertain service relationship: legitimate parent or trace context leakage?

`caviar-calculate-risk` is from a **different service** than CAVIAR Haystack API, and the expected root of the trace should be `GET EquityVol/GetEquityDividend`. There are two possible explanations for why `caviar-calculate-risk` appears at the root:

#### Scenario A — Legitimate distributed trace (upstream caller)

`caviar-calculate-risk` (in Service X) is genuinely an upstream entry point that initiates a chain of calls which eventually reaches CAVIAR Haystack API's `GET EquityVol/GetEquityDividend`. The `traceparent` / `tracestate` header is correctly propagated through each hop. The 135,224 items are the cumulative spans from every service in that chain.

In this scenario, the root cause is **trace size + the maxTraceItems cap**, and the remediation is to raise the cap and fix the instrumentation generating 135k items.

#### Scenario B — Trace context leakage (unrelated transactions sharing a trace ID)

`caviar-calculate-risk` and `GET EquityVol/GetEquityDividend` are **not in the same logical call chain** but share the same `trace.id` by accident. The 135,224 items are many unrelated requests that have been incorrectly stitched together under one trace. This is a serious instrumentation defect and the primary thing to fix.

Common causes of trace context leakage:

| Cause | Description |
|---|---|
| **Message queue context not reset** | A queue consumer (e.g., Kafka, RabbitMQ) picks up the `traceparent` header from the message envelope and continues the producer's trace instead of starting a fresh transaction |
| **Thread pool / executor inheritance** | A background thread inherits the trace context from the request that submitted the task, causing all async work to appear as children of that request |
| **HTTP connection reuse** | A persistent HTTP client reuses a connection that still carries old trace headers in memory |
| **Application code manually propagating trace ID** | Code that reads and forwards `trace.id` or `traceparent` across unrelated request boundaries |
| **APM agent misconfiguration** | Agent configured to always continue incoming trace context without checking whether it belongs to the current logical request |

---

## How to Diagnose: Which Scenario Applies?

Run these queries in Kibana Discover (KQL) to determine whether the relationship is legitimate or a leak.

**Step 1 — Get the trace ID of the problematic sample**

In the APM UI, click the `GET EquityVol/GetEquityDividend` trace sample → **Open in Discover**. Note the `trace.id` value.

**Step 2 — Check the parent relationship**

In Discover, filter on that `trace.id` and look for the `GET EquityVol/GetEquityDividend` document specifically:

```
trace.id: "<your-trace-id>" AND transaction.name: "GET EquityVol/GetEquityDividend"
```

Expand the document and check the `parent.id` field. Then search for a span or transaction with that `parent.id` as its `span.id` or `transaction.id`:

```
trace.id: "<your-trace-id>" AND (span.id: "<parent.id>" OR transaction.id: "<parent.id>")
```

If the parent document belongs to the `caviar-calculate-risk` transaction (or its spans) **and you can trace an unbroken chain** from `caviar-calculate-risk` down to `GET EquityVol/GetEquityDividend` → **Scenario A**.

If the parent document does not exist, belongs to a completely unrelated request, or `GET EquityVol/GetEquityDividend` has no `parent.id` at all → **Scenario B**.

**Step 3 — Check the service map**

In the APM UI, go to **Service Map**. Does it show a direct or transitive arrow from the service containing `caviar-calculate-risk` to `CAVIAR Haystack API`? If no connection exists → strongly indicates Scenario B.

**Step 4 — Check how CAVIAR Haystack API receives its requests**

Look at the incoming transport for `GET EquityVol/GetEquityDividend`. Is it:
- A direct HTTP/gRPC call from another service? → Check that the calling service is the one that contains `caviar-calculate-risk`.
- A message queue consumer? → Very likely Scenario B; the message was produced by `caviar-calculate-risk` with its trace context embedded, and the consumer incorrectly continued that context.

---

## `xpack.apm.ui.maxTraceItems` — Complete Reference

### Default value

**5,000** — confirmed from [Kibana source (`x-pack/solutions/observability/plugins/apm/server/index.ts`)](https://github.com/elastic/kibana/blob/main/x-pack/solutions/observability/plugins/apm/server/index.ts):

```ts
maxTraceItems: schema.number({ defaultValue: 5000 })
```

This cluster is set to **2,000**, which is below the default. This makes the truncation problem worse than out-of-box behaviour.

### Acceptable values

Any positive integer — the schema imposes no enforced minimum or maximum. However there is a **practical hard ceiling**:

**Maximum safe value without additional configuration: 10,000**

This is imposed by Elasticsearch's default `index.max_result_window` of 10,000. Setting `maxTraceItems` above 10,000 will cause Elasticsearch to reject the trace query with a `Result window is too large` error unless `index.max_result_window` has been explicitly raised on the APM indices. Batched pagination for this query was tracked as a future improvement in [Kibana issue #164937](https://github.com/elastic/kibana/issues/164937) but was not the default behaviour in most 8.x/9.x releases.

| Value | Notes |
|---|---|
| 2,000 | Your current value — below default, worsens the problem |
| 5,000 | Shipped default — safe starting point |
| 10,000 | Maximum safe value without raising `index.max_result_window` |
| > 10,000 | Requires raised `index.max_result_window` AND Kibana version with batched ES pagination |

### Why is the behaviour inconsistent? (works sometimes, fails other times)

There is **no ratio** — it is a binary outcome based on whether your specific transaction appears within the first N chronological items of the trace sample the UI happens to select.

When a trace is fetched, Kibana queries Elasticsearch for up to `maxTraceItems` transactions and spans, **sorted by timestamp ascending**. It takes the first N items. Errors are fetched separately and do not count against the limit ([PR #56111](https://github.com/elastic/kibana/pull/56111)).

| Trace sample selected | Outcome |
|---|---|
| Small trace (< 2,000 items) not involving `caviar-calculate-risk` | ✅ All items fit — waterfall renders correctly |
| The 135,224-item `caviar-calculate-risk` trace | ❌ Only root + immediate children appear — `GET EquityVol/GetEquityDividend` is truncated |

The APM UI rotates through trace samples as you navigate the latency distribution. The inconsistency you observe directly tracks which sample the UI is currently presenting.

Additionally, as noted in [issue #120464](https://github.com/elastic/kibana/issues/120464), if an **intermediate parent span is truncated**, its entire subtree also disappears from the waterfall, compounding the visual breakage.

### Impact of setting `maxTraceItems` too high

| Risk | Detail |
|---|---|
| **Elasticsearch query rejection** | Above 10,000 ES rejects the query unless `index.max_result_window` is raised |
| **Browser memory / freeze** | The waterfall renders as a DOM tree; 10,000 nodes is already very heavy |
| **Kibana server memory** | Full payload loaded into Kibana's Node.js process before being sent to the browser |
| **Slow UI response** | Large ES queries compete with other Kibana queries under heavy APM ingest |
| **No upside above trace size** | Setting it to 50,000 gives zero benefit over 10,000 if ES caps it at 10,000 anyway |

---

## Remediation — Ordered by Lowest Effort First

### Step 0 — Fix the version 9.3 known bug (upgrade, ~30 min)

If on **9.3.0 or 9.3.1**, upgrade to **9.3.2** (or 9.4.0) to fix the `timestamp.us` waterfall rendering bug. Run the detection query above first to confirm whether you are affected. If you cannot upgrade immediately, apply the `traces-apm@custom` workaround described in the Version 9.3 Known Bug section.

### Step 1 — Immediate workaround (no config change required)

- Click **"View full trace"** or **"Open in Discover"** on the trace sample panel. This takes you directly to the Elasticsearch document for the specific transaction, bypassing the truncated waterfall. You will see the correct transaction details.
- Alternatively, navigate the latency distribution arrows to pick a different trace sample — one from a smaller trace that does not involve `caviar-calculate-risk`.

### Step 2 — Restore the default (Kibana restart only, ~5 min)

Your current value of 2,000 is below the shipped default of 5,000:

```yaml
# kibana.yml (self-managed)
# or Kibana user settings on Elastic Cloud
xpack.apm.ui.maxTraceItems: 5000
```

Restart Kibana. This resolves the issue for any trace smaller than 5,000 items and correctly benchmarks the problem against the designed default.

### Step 3 — Raise to the safe ceiling (Kibana restart only, ~5 min)

If the 135k-item trace still fails after Step 2:

```yaml
xpack.apm.ui.maxTraceItems: 10000
```

Do **not** go above 10,000 without first verifying your Kibana version supports batched ES pagination for this query ([issue #164937](https://github.com/elastic/kibana/issues/164937)).

### Step 4 — Diagnose the service relationship (no config change, ~30 min)

Run the diagnostic queries in the section above to determine whether Scenario A or Scenario B applies. This determines whether the next steps are a configuration fix (Scenario A) or an instrumentation/architecture fix (Scenario B).

### Step 5a — If Scenario A (legitimate trace): fix the span count

Check the APM agent `transaction_max_spans` setting on the `caviar-calculate-risk` service (default: **500 per transaction**). If raised, lower it:

```yaml
transaction_max_spans: 500   # Java, Python; Node.js: transactionMaxSpans
```

Agent docs: [Java](https://www.elastic.co/guide/en/apm/agent/java/current/config-core.html#config-transaction-max-spans) · [Python](https://www.elastic.co/guide/en/apm/agent/python/current/configuration.html#config-transaction-max-spans) · [Node.js](https://www.elastic.co/guide/en/apm/agent/nodejs/current/configuration.html#transaction-max-spans)

Then investigate the span type producing 100k+ items (per-row DB calls, per-item HTTP, per-message queue publishes) and batch those operations.

### Step 5b — If Scenario B (trace context leakage): fix the propagation

- **Message queue consumers:** Start a new transaction (`apm.startTransaction(...)`) inside the consumer instead of continuing the producer's trace context. Do not forward the `traceparent` header from the message envelope into the new transaction.
- **Thread pools / async executors:** Ensure background threads clear or create their own trace context rather than inheriting the submitter's.
- **HTTP client reuse:** Verify the HTTP client is not caching `traceparent` headers between requests.
- **If external context is unavoidable:** Some APM agents support an `ignore_incoming_trace_ids` or equivalent option to refuse external trace contexts from untrusted sources.

---

## References

| Resource | Link |
|---|---|
| APM Settings in Kibana (official docs) | [elastic.co · apm-settings-kb](https://www.elastic.co/guide/en/kibana/current/apm-settings-kb.html) |
| Kibana source — maxTraceItems defaultValue: 5000 | [github.com · kibana/apm/server/index.ts](https://github.com/elastic/kibana/blob/main/x-pack/solutions/observability/plugins/apm/server/index.ts) |
| Known bug fix — timestamp.us mapping as long | [github.com · elasticsearch#143173](https://github.com/elastic/elasticsearch/pull/143173) |
| Elastic APM Known Issues page | [elastic.co · apm/known-issues](https://www.elastic.co/docs/release-notes/apm/known-issues) |
| PR #56111 — Fixes maxTraceItems in waterfall & error queries | [github.com · kibana/pull/56111](https://github.com/elastic/kibana/pull/56111) |
| Issue #164937 — Trace waterfall pagination for big traces | [github.com · kibana/issues/164937](https://github.com/elastic/kibana/issues/164937) |
| Issue #120464 — Missing trace items shouldn't break waterfall | [github.com · kibana/issues/120464](https://github.com/elastic/kibana/issues/120464) |
| Issue #118282 — Add more info to the "exceeds max" callout | [github.com · kibana/issues/118282](https://github.com/elastic/kibana/issues/118282) |
| Distributed tracing — W3C TraceContext in Elastic APM | [elastic.co · blog/elastic-apm-adopts-w3c-tracecontext](https://www.elastic.co/blog/elastic-apm-adopts-w3c-tracecontext) |

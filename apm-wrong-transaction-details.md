# APM: Wrong Transaction Details Displayed in Kibana

## Summary

In the Kibana Observability APM app, opening `CAVIAR Haystack API → Transactions → GET EquityVol/GetEquityDividend` and picking a trace sample renders a waterfall whose root/entry transaction is `caviar-calculate-risk` — not the selected `GET EquityVol/GetEquityDividend`. A banner on the waterfall reports:

> The number of items in this trace is **135,224** which is higher than the current limit of **2,000**. Please increase the limit via `xpack.apm.ui.maxTraceItems` to see the full trace.

The panel also shows an "Incomplete trace" tag.

## Cause

Two things are happening at once.

### 1. Expected behavior — the trace-samples panel shows the whole distributed trace

The Trace samples section on the Transaction detail page renders the entire distributed trace that the selected transaction participated in, rooted at that trace's first transaction. Because `GET EquityVol/GetEquityDividend` is a downstream call executed inside a larger `caviar-calculate-risk` request, the waterfall correctly opens at `caviar-calculate-risk`. The selected transaction is meant to appear further down the waterfall as one of its children.

### 2. Root cause — the trace is truncated by `xpack.apm.ui.maxTraceItems`

Kibana's APM UI caps the number of items (transactions + spans) it will fetch per trace at `xpack.apm.ui.maxTraceItems` (default 5,000; this cluster is set to 2,000). When a trace has 135,224 items, the query returns only the earliest 2,000 by timestamp — the root transaction and its immediate children — and drops everything else. The selected `GET EquityVol/GetEquityDividend` transaction is one of the truncated items, so it never renders in the waterfall. That is why the details on screen look like they belong to a different transaction.

**So yes — the 135,224 vs 2,000 warning is directly causing the observed behavior.**

## Remediation

### Short-term (unblock the user immediately)

- Click **View full trace** or **Open in Discover** to jump straight to the specific transaction document instead of relying on the truncated waterfall.
- Pick a different trace sample from the latency distribution — one whose root is not the oversized `caviar-calculate-risk` request. That will give a normal waterfall for `GET EquityVol/GetEquityDividend`.

### Medium-term (raise the UI limit)

Increase `xpack.apm.ui.maxTraceItems` in Kibana. Locations:

- **Self-managed:** `kibana.yml`
- **Elastic Cloud:** Deployment → Kibana → Edit user settings
- **Serverless / Observability project:** Stack Management → Advanced Settings

```yaml
xpack.apm.ui.maxTraceItems: 20000
```

Restart Kibana. Recommended ceiling is 10k–20k — pushing this into the hundreds of thousands makes the trace query heavy and can slow down or OOM Kibana / the browser. Even at 20k the 135k-item trace still won't fully render, but the selected transaction should appear.

### Long-term (fix the instrumentation)

A single trace with 135,224 items is almost always an instrumentation defect, not a Kibana one. Investigate `caviar-calculate-risk` for:

- **Per-iteration spans in a loop** — per-row DB calls, per-item HTTP calls, per-message queue publishes. Batch the operation or wrap the loop in a single parent span.
- **Trace-context leakage across requests** — background workers, async jobs, or message consumers that continue an incoming trace context instead of starting a new transaction (`apm.startTransaction`). Independent requests get stitched into one giant trace.
- **`transaction_max_spans` raised too high in the APM agent** — the default of 500 per transaction is there for a reason. If it was bumped, lower it back.
- **Recursive fan-out** — `caviar-calculate-risk` calling itself or fanning out to thousands of downstream calls. Consider downstream sampling or breaking the workload into separate traces.

Recommended next step: from Service inventory, open `caviar-calculate-risk`, pick one of its transactions, and inspect the span breakdown / span type to identify which span type (db, http, external, custom) is producing tens of thousands of items. Fixing the source removes the need to keep raising `maxTraceItems`.

## References

- Kibana setting: [`xpack.apm.ui.maxTraceItems`](https://www.elastic.co/guide/en/kibana/current/apm-settings-kb.html)
- Elastic APM agent config: `transaction_max_spans`

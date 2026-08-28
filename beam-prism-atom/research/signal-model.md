# Beam→Atom signal model — research & proposal draft

Status: DRAFT for discussion with Neil, 2026-08-28.
Inputs: four parallel research reports (`history://BeamSignals`, `history://AtomIngestQuery`, `history://AxiomAPL`, `history://PriorArt`), full citations there.

## Ground truth (what the research established)

### Beam already produces most of the taxonomy
15 typed `EventEnvelope` kinds (`{id, timestamp, host, kind, qos, payload}`, snake_case). The important ones by class:

- **Roll-ups exist**: 30s pipeline windows emit metric series keyed by name/unit/attrs — count/sum/min/max, **mergeable exponential histogram bins**, p50/p95/p99, ≤4 exemplars, cap 256 series (`beam-pipeline/src/aggregation.rs`). GPU obs → `beam.gpu.process.used_memory` / `.sm.utilization` / `.memory.utilization` with GPU dims.
- **Change side-band exists**: `change_event` = category, stable key, `before`/`after`, provenance evidence; P0 never-shed; deterministic materialized-state diff (not raw observation), source-error suppression avoids false removals.
- **Health-ish but no anomalies**: adapter states (running/degraded/failed…), sensor drop counters, GPU cycle degraded→running recovery. No detector, no health score; world-model synthesis explicitly deferred.
- **Evidence**: attribution nodes/edges/submissions (P0, never summarize), kernel exec/exit aggregates, system/journald/audit events. Everything also lands in the local flight-recorder ring (1GiB NDJSON) for promotion.

**Transport gap**: exactly one `capture_endpoint`; NDJSON goes to fixed `/v1/events`, OTLP is logs-first. **No dataset routing** — multi-dataset needs a routing field, destination-aware WAL partitioning or per-dataset workers. Not config-only.

### Atom is a pure passthrough — and OTLP metrics is a dead end
- Ingest streams raw NDJSON/JSON/CSV (+gzip/zstd) and OTLP **logs/traces** to Metal EventDB; APL forwarded untouched; **no atom-side aggregation**. MPL/metrics endpoints 404 by design.
- Cloud Axiom routes OTLP metrics to a separate MetricsDB dataset kind that **rejects APL** and **skips histograms** — so even upstream, OTLP metrics ≠ APL-queryable. Events-as-metrics is the only path that works everywhere (atom now, axiomhq/axiom later).
- EventDB flattens nested JSON to dot-path fields (key ≤200B); `x-axiom-object-fields` preserves named fields as maps; `_time` default, custom timestamp-field supported.
- Datasets: ≤80-char names, per-dataset retention/trim/tokens — a `beam.*` split layout is valid and independently governable.
- **Atom console is management-only** — no query editor, no charts. "Showing" signals = portal (Grafana via APL projections) or new console surface. Separate scope decision.

### APL is the arbiter of "readable comfortably"
- `summarize` supports avg/count/dcount/sum/min/max/**percentile (hybrid: exact ≤64 values, then DDSketch @ 1% relative accuracy)**/topk/variance/stdev/histogram/rate/make_list/make_set; `bin`/`bin_auto(_time)` for time bucketing.
- **APL cannot merge a serialized digest carried in an event.** Precedent inside axiomhq/axiom: **Stitch** emits per-minute wide event rows with scalar roll-ups + DDSketch-compatible sparse-bin arrays + exemplars; consumers `make_list` the bins and merge **outside** APL. That is the house style for pre-aggregated events.
- Discipline: low-cardinality dimensions, bounded key sets, flattened dot paths by default.

### Prior art
- OTel data model: delta temporality for stateless edges; 10s→60s reaggregation is the documented norm; windows carry start+end timestamps; resets signaled by changed start time.
- DDSketch = formal relative-error + mergeable; scalars + fixed percentiles suffice **only when you never re-aggregate across windows/hosts**.
- State-transition schemas converge (K8s Events, Datadog monitor transitions, CloudEvents): `state, prior_state, reason (machine-short), severity, subject, evidence, dedup/series key, count/first/last`.

## Proposal

### Signal taxonomy — five classes, five datasets

| Dataset | Content | Source (beam today) | Cadence | QoS |
|---|---|---|---|---|
| `beam.metrics` | Per-window wide metric events: `_time`=window end, `window.start`, `metric.name/unit`, dims, `count/sum/min/max`, `p50/p95/p99`, optional `bins[]` (exp-histogram, Stitch-style) | pipeline aggregates, GPU/DCGM series | 30s (align epoch; delta windows) | P1/P2 |
| `beam.changes` | `change_event` as-is: category, key, before/after, provenance | inventory diff | on change | P0 |
| `beam.state` | **New normalized `condition_event`**: subject{kind,id}, state, prior_state, since, reason, severity, evidence refs. Covers adapter/GPU/sensor lifecycle today; **anomaly=enter-abnormal, recovery=return-to-normal tomorrow** — same schema, detectors slot in later without wire changes | adapter_status, GPU cycle status, sensor lifecycle (normalized) | on transition | P0 |
| `beam.inventory` | Compact snapshots + host_baseline ("what is this host now"; latest via `arg_max`) | inventory_snapshot, host_baseline | 5m + boot | P2 |
| `beam.events` | Evidence: attribution graph (never summarized), kernel aggregates/exemplars, system/audit events, self_telemetry | remaining kinds | live/30s | mixed |

Answer to the side-band question: **yes, and it already half-exists.** `change_event` is the change side-band; the missing piece is one new `condition_event` kind that normalizes today's scattered lifecycle/degraded/recovery signals and becomes the anomaly/recovery channel when detectors land. Known-schema timestamped JSON, exactly as you framed it.

### Wire language: native NDJSON wide events on the Axiom ingest API
- Beam gains an `axiom+http(s)` endpoint scheme: `POST {base}/v1/ingest/{dataset}`, bearer token, NDJSON, gzip/zstd, `timestamp-field` mapping from envelope timestamp, per-class dataset routing, existing idempotency keys preserved end-to-end.
- **Prism compatibility falls out for free**: prism's listener is the *same* API shape (`/v1/ingest/{dataset}` + bearer). Beam→atom direct and beam→prism→atom differ only in base URL + token. Prism optional, always compatible, zero beam-side branching.
- OTLP stays as the third-party interop path (logs/traces to atom's OTLP endpoints), not the primary beam→atom language. OTLP metrics is unsupported by atom and histogram-lossy upstream — rejected as primary.
- Flatten payloads to dot-path field names by design. `before`/`after` in change events are shape-unbounded → send via `x-axiom-object-fields` (map fields) or as JSON strings to protect field cardinality. Decide at schema-pinning time.

### Cadence & merge semantics
- 30s edge windows (existing), epoch-aligned, `window.start`/`window.end`, delta temporality, reset = new start time (OTel convention).
- Scalars answer single-host dashboard queries natively in APL (`avg`=sum/count, max-of-max, percentile-of-p95 as approximation). Cross-host/cross-window exact percentiles use the `bins[]` arrays merged Stitch-style by the reading surface (portal/console), not APL.
- Cardinality budget per class: dims limited to host, gpu.uuid/index, workload.id/kind, adapter.id, unit; PIDs live only in exemplars.

## Open decisions for Neil

1. **D1 dataset split** — five `beam.*` datasets (proposed; independent retention/tokens, cheap queries) vs one dataset + `kind` discriminator. Recommend split.
2. **D2 histogram bins** — ship `bins[]` in v1 (bigger events, exact cross-host percentiles later) or scalars-only v1 with bins behind config? Recommend bins-on, they're already computed.
3. **D3 before/after encoding** in `beam.changes` — object-fields map vs JSON string vs raw flatten. Cardinality risk lives here.
4. **D4 condition_event schema** — sign off field set (subject, state, prior_state, since, reason, severity, evidence).
5. **D5 where signals get *shown*** — atom console today can't chart anything. Grafana-portal APL projections first? New atom console pages? Both are atom-repo scope, not beam.
6. **D6 anomaly detection v1 scope** — model above deliberately ships the *channel* without detectors (threshold/state transitions only). Detectors (baseline deviation, world-model) become pure beam-internal work later. Confirm that sequencing.
7. **D7 beam's "prism relay" docs** — beam DECISIONS still describe a Go fleet relay named prism; reconcile with Rust prism reality when we plan the prism leg.

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

### Signal taxonomy — REVISED (2026-08-28, after discussion): two planes, two managed datasets

Supersedes the initial five-dataset sketch. Neil's push: an envelope should collapse the count; beam also emits *normal* OTel alongside its managed picture.

**OTel plane (shared datasets):** beam emits native OTLP logs/traces into the same datasets other sources use — journald/audit, kernel exemplars as logs with trace/span context where available. Note this changes beam's OTLP path: today it wraps its own envelope in every log record; the plane split requires native OTel semantics there instead.

**Beam plane (managed datasets):** only one boundary is load-bearing by EventDB/APL criteria (retention, scan volume, schema shape) — high-volume uniform periodic rows vs low-volume discrete statements:

| Dataset | Content | Cadence | QoS |
|---|---|---|---|
| `beam.metrics` | Uniform window rows: `_time`=window.end, `window.start`, `metric.name/unit`, `dims.*`, `agg.{count,sum,min,max,p50,p95,p99}`, `bins[]` (exp-histogram, Stitch-style) | 30s epoch-aligned, delta | P1/P2 |
| `beam.signals` | One envelope for everything discrete — changes, conditions/anomalies/recoveries, inventory, attribution. All are the same sentence: *"at T, subject S is/became X, because R, here's evidence."* | on change/transition + low-cadence inventory heartbeat | P0 |

`beam.signals` envelope:

```
_time, host.name, host.boot_id
signal.kind      change | condition | inventory | attribution
subject.category package | unit | listener | container | gpu | adapter | sensor | workload | node | edge
subject.key      stable identity ("openssl", "sshd.service", "tcp/0.0.0.0:443", gpu.uuid…)
state, prior_state, since, reason, severity
evidence.ids[]   envelope ids → flight recorder / OTel logs drill-down
```

Field discipline, three tiers — the "not a map dump" answer:
1. envelope — always present, always typed;
2. promoted well-knowns per category — the queryable fields, hand-picked and bounded (`package.version.before/after`, `listener.port`, `gpu.index`, …);
3. residue — arbitrary before/after shapes as JSON strings (`change.before_json`): greppable forensics, zero field-explosion risk.

**Row orientation kills the map problem.** Inventory snapshots as maps would flatten into thousands of sparse fields. Instead: one row per entity in the envelope (`signal.kind=inventory`, `subject.category=package`, `subject.key=openssl`, `package.version=3.2.1`). Latest host state = `summarize arg_max(_time, *) by subject.key`; a change is the same row shape with `prior_state`/`before` filled. Snapshots drop to a reconciliation heartbeat (~hourly) since changes carry deltas.

The former `beam.events` class dissolves: evidence is either OTel logs (shared plane) or attribution rows (`beam.signals`). Kernel exec/exit aggregates are counts → `beam.metrics`.

Cost of the merge, honestly: changes/inventory share retention+tokens with conditions (acceptable — all long-retention P0-ish), and `signal.kind` filters replace dataset selection (trivial scan overhead at side-band volumes).

Anomaly/recovery answer unchanged: `condition` rows in `beam.signals` — anomaly = enter-abnormal, recovery = return-to-normal; detectors slot in later with zero wire changes.


### Wire language: native NDJSON wide events on the Axiom ingest API
- Beam gains an `axiom+http(s)` endpoint scheme: `POST {base}/v1/ingest/{dataset}`, bearer token, NDJSON, gzip/zstd, `timestamp-field` mapping from envelope timestamp, per-class dataset routing, existing idempotency keys preserved end-to-end.
- **Prism compatibility falls out for free**: prism's listener is the *same* API shape (`/v1/ingest/{dataset}` + bearer). Beam→atom direct and beam→prism→atom differ only in base URL + token. Prism optional, always compatible, zero beam-side branching.
- OTLP stays as the third-party interop path (logs/traces to atom's OTLP endpoints), not the primary beam→atom language. OTLP metrics is unsupported by atom and histogram-lossy upstream — rejected as primary.
- Flatten payloads to dot-path field names by design. `before`/`after` in change events are shape-unbounded → send via `x-axiom-object-fields` (map fields) or as JSON strings to protect field cardinality. Decide at schema-pinning time.

### Cadence & merge semantics
- 30s edge windows (existing), epoch-aligned, `window.start`/`window.end`, delta temporality, reset = new start time (OTel convention).
- Scalars answer single-host dashboard queries natively in APL (`avg`=sum/count, max-of-max, percentile-of-p95 as approximation). Cross-host/cross-window exact percentiles use the `bins[]` arrays merged Stitch-style by the reading surface (portal/console), not APL.
- Cardinality budget per class: dims limited to host, gpu.uuid/index, workload.id/kind, adapter.id, unit; PIDs live only in exemplars.

## Decisions (resolved 2026-08-28 with Neil)

1. ~~D1 dataset split~~ — two managed datasets (`beam.metrics`, `beam.signals`) + shared OTel logs/traces datasets.
2. ~~D2 histogram bins~~ — ship `bins[]` in v1. Percentiles don't compose; scalars-only makes fleet-wide p95 an approximation, bins let readers merge for exact answers. Beam already computes them; dropping is irreversible loss, ignoring is free.
3. ~~D3 before/after encoding~~ — promoted well-knowns (curated per-category typed columns, e.g. `package.version.before/after`, `listener.port`) + full objects as JSON-string residue (`change.before_json`). Generic flattening rejected: every distinct shape permanently pollutes dataset schema.
4. ~~D4 condition envelope~~ — approved as drafted (subject, state, prior_state, since, reason, severity, evidence).
5. ~~D5 read surface~~ — Grafana + **Axiom datasource plugin** against atom's APL endpoint (real APL). Not grafana-portal Loki/Tempo projections (forces logs/traces UX).
6. ~~D6 sequencing~~ — nail beam↔atom 100% first; add prism support after. No detectors in v1; `condition` channel ships with state-transition sources only.
7. ~~D7~~ — prism-relay doc reconciliation deferred to the prism leg.

## Atom-side requirements (added 2026-08-28)

Atom diverges from Axiom deliberately: natural dataset lifecycle + honest, constrained ingest kinds. Evidence: `history://AtomOtelGap`, `history://OverstreamOtel`. Overstream (overstreamhq/overstream, `worker-research` branch) is Neil's testing ground for Atom/Axiom's future; work migrates into atom bit by bit.

### A1 — auto-create datasets on ingest
Today `POST /v1/ingest/{unknown}` → 404 after name validation + auth, before EventDB (`ingestapi/handler.go:147-170`). Metal `CreateDataset` = idempotent `PUT /datasets/{fqdn}` (`dbclient/datasets.go:407-438`) → lazy create-on-first-ingest is race-safe: resolve-or-insert Postgres org/name→FQDN mapping, ensure in Metal, forward. (Overstream instead pre-creates a fixed logical set at startup with cached ensure+retry — atom goes further: as-needed.) RESOLVED: auto-create requires token with both `create` and `ingest` on the dataset name; audit row on create.

### A2 — bare OTel → auto-created default datasets
Today bare `/v1/logs` / `/v1/traces` without dataset header → 400 (`handler.go:113-146`). New: route to per-kind default datasets, auto-created via A1. RESOLVED: default names `otel-logs` / `otel-traces`.

### A3 — dataset kinds, v2-only
Atom has NO kind concept: no column in `atom.datasets` (migration 0001), create API accepts/echoes hardcoded `axiom:events:v1` (`datasetsapi.go:22-24,219-226`), Metal client create carries no kind. Add: kind column, creation-time validation, per-kind ingest enforcement. Kind is *the* mechanism for atom's constraints-Axiom-doesn't-have:
- `events` — free-form v2 ingest;
- `otel-logs` / `otel-traces` — only OTLP accepted, projected (A4);
- `beam-metrics` / `beam-signals` — envelope-validated (signal.kind/subject.* required, window fields, etc.).
Exposed via v2 dataset API only; v1 keeps legacy hardcoded response for portal compat. RESOLVED kind strings: `axiom:events:v2`, `axiom:otel-logs:v2`, `axiom:otel-traces:v2`, `axiom:beam-metrics:v2`, `axiom:beam-signals:v2` — deliberately `axiom:*`, since Axiom adopts these kinds in time (atom makes the mistakes first). Kind difference makes client query adjustment trivial.

### A4 — OTLP projection (lift from overstream)
Atom streams OTLP raw to Metal (byte-identical, `handler_test.go:374-429`); row shape is Metal's axiom-derived dotted sprawl. Adopt overstream's decode-once → typed canonical rows (`internal/gateway/decode.go`, `internal/axiom/ingest.go:149-218`):
- resource attrs `resource.<key>`; record attrs unprefixed; `_time` (receipt), `receipt_time`, `source_time`, `signal`;
- spans: hex `trace_id`/`span_id`/`parent_span_id`, `name`, `kind`, `duration_ns`, `status`, `status_message`;
- logs: `severity` int, canonical `body`, optional trace correlation.
Fix two known overstream-prototype gaps during the lift (docs intend both): scope name/version currently discarded; `severity_text` missing. Consequence: atom stops raw-streaming for OTLP content types — buffer→decode→project→NDJSON; seam between `ingestOptions` and `EventDB.Ingest`.

### Parked overstream liftables (later epics, not now)
Strict typed gate with quarantine + `_failures`; natural keys/fingerprints/actor facets; request-wide size limits; token env-scoping; epoch-based schema evolution (logical→physical dataset registry); projection specs/fidelity raise-ups; receipts; console patterns.

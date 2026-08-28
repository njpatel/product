# E2E acceptance evidence — beam↔atom signals v1

Run: 2026-08-28, host `adipurush`. All six plan acceptance steps pass.

## Stack (all live at capture time)

| Component | Detail |
|---|---|
| EventDB (real, axiom `cmd/db`) | `serve -role default :55081` + `run-delta-cache :55082`, `-dataset-tracker-local-mode`, postgres:55501, redis:55502 |
| atom | integration branch build, `:55080`, postgres:55503 |
| Grafana 13.1.3 + axiomhq-axiom-datasource v0.7.0 | `:55300`, provisioned `apiHost`/`edgeURL` → atom, xaat token |
| beam node | integration branch build, `axiom+http://127.0.0.1:55080`, fleet profile, 5s/10s intervals, no root, no eBPF sensor |

## Step 1+2 — auto-created datasets, correct kinds, zero pre-setup

```
GET /v2/datasets →
  beam.metrics  axiom:beam-metrics:v2   (auto-created on beam's first ingest, X-Axiom-Dataset-Kind)
  beam.signals  axiom:beam-signals:v2   (same)
  otel-logs     axiom:otel-logs:v2      (auto-created by bare POST /v1/logs)
```
Admission proof: flat dotted key rejected `400 record 1: key "host.name" must be sent as nested objects…`; missing `subject.key` rejected naming record+field; ingest-only token gets 404 (needs `create`).

## Step 3 — host changes → queryable change rows (~10s latency)

Full initial inventory shipped by the (fixed) node: **2173 package / 934 unit / 375 listener / 56 container / 8 gpu / 2 node** rows. Live scenario changes:

```
APL: ['beam.signals'] | where ['signal.kind'] == 'change' and ['subject.category'] in ('listener','container')
13:48:28  listener   tcp://0.0.0.0:55779#pid=2257185      (python http.server started)
13:48:28  listener   tcp://127.0.0.1:55888#inode=…        (nginx container port removed)
13:48:28  container  88085cdf519d…                        (docker rm nginx)
```
(+20 natural systemd unit-state changes captured in the first minutes.)

## Step 4 — metrics as windowed rows, charted via real APL in Grafana

`grafana-apl-p95-chart.webp`: Grafana Explore, APL editor, `['beam.metrics'] | where ['metric.name'] == 'beam.host.memory.available_bytes' | summarize avg(['agg.p95']) by bin(_time, 1m)` rendering a live time series. Rows carry `agg.{count,sum,min,max,p50,p95,p99}` + `bins` + exemplar envelope ids. Plugin Save&Test: `{"message":"Configuration is valid and ready to use","status":"OK"}` (atom's 422 empty-APL probe).

## Step 5 — anomaly → recovery condition pair

nvidia-smi failure injected via PATH shim for ~35s:

```
APL: ['beam.signals'] | where ['signal.kind'] == 'condition'
13:48:23  adapter  nvidia_smi_procfs/process_collection  running→degraded
          reason: GPU process collection failed: nvidia-smi --query-gpu=… failed
13:48:58  adapter  nvidia_smi_procfs/process_collection  degraded→running
          reason: GPU process collection recovered
```
`prior_state` correct on both rows — the condition channel works exactly as designed: anomaly = enter-abnormal, recovery = return-to-normal, no detector required.

## Step 6 — bare OTLP → clean projection in auto-created default dataset

```
POST /v1/logs (no dataset header) → 200, dataset otel-logs created
APL project: _time, body, severity_text, resource.service.name, scope.name, scope.version
  bpa-e2e / INFO / "beam-prism-atom e2e scenario log line" / scope bpa.scenario 1.0
  + beam's own native-OTel plane records (service.name=beam-node, scope=beam.pipeline)
```
`scope.*` retained and `severity_text` present — the two overstream-prototype gaps fixed. All fields addressable **unescaped** in APL.

## Integration discoveries (fixed during slice 5, committed on integration branches)

1. **Wire contract amendment — nested JSON (all senders).** EventDB stores literal-dot flat keys as escaped fields (`['signal\.kind']`), poisoning APL. Rows now ship nested; EventDB flattens to clean dot-path names. beam nests at WAL enqueue; atom's OTLP projection nests; beam-* admission validates nested paths and rejects top-level dotted keys.
2. **atom APL parser bug**: every `['…']` bracket group was treated as a dataset reference, so field refs (`['signal.kind']`) 404'd queries. Fixed: dataset refs only in source position (start / after `union`).
3. **beam startup deadlock**: main queued the ~3.5k-record initial inventory into the 256-cap event channel before the runtime loop started — permanent futex park, no followers. Initial inventory now emitted from the inventory runner's thread.
4. Grafana plugin addenda all exercised: 422 empty-APL probe, `/v1/datasets/_fields`, kind in `/v2/datasets`.

## Follow-ups (not blocking, for future slices)

- Dataset DELETE can half-fail (EventDB delete 503 → Metal side gone, registry row left → ingest 404s). Needs a reconciliation/tombstone story.
- `beam serve node` without `--interval` is a silent single-pass run — surprising default; consider requiring an explicit `--once`.
- Change rows for listener/container add/remove carry null `state`/`prior_state`; per-category state vocabulary worth defining.
- beam Cargo.lock drift in the `njpatel-init` worktree noted at survey time (unrelated).
- OTLP traces plane from beam not yet exercised (no span sources in beam today).

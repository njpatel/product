# Plan: beam↔atom signals v1

Epic: `beam-prism-atom` · Status: AGREED PENDING NEIL SIGN-OFF · 2026-08-28
Design basis: `research/signal-model.md` (all decisions resolved), `research/` scout reports.

## Goal

Beam sends its full signal picture directly to atom in the agreed two-plane model; the result is queryable with real APL from Grafana via the Axiom datasource plugin. Prism is **out of scope** (added after beam↔atom is 100%; wire compatibility is preserved by design since prism speaks the same ingest API).

**E2E acceptance scenario** (the epic is done when this runs):
1. Metal EventDB + Postgres + atom + Grafana(+Axiom plugin) up; beam running on a host.
2. Datasets `beam.metrics`, `beam.signals`, `otel-logs` appear in atom **without any pre-setup** (auto-created, correct kinds).
3. Install/remove a package on the host → `beam.signals` change row (promoted fields + JSON residue) queryable via APL in Grafana within ~1min.
4. Host/GPU metrics visible as windowed rows in `beam.metrics`; a Grafana panel charts p95 over time via APL.
5. Stop/start a beam adapter (or degrade GPU collection) → `condition` transition rows (abnormal → recovery) in `beam.signals`.
6. Bare OTLP logs POST to atom → lands in auto-created `otel-logs` with the clean projection (scope + severity_text present, resource.* prefixed).

## Non-goals

Prism leg; anomaly detectors (channel only); atom console charting; MetricsDB anything; overstream liftables beyond OTLP projection (parked list in research doc); multi-node fleets.

## Contracts (pinned up front — all slices code against these)

### C1 — kind strings
`axiom:events:v1` (legacy, unchanged) · `axiom:events:v2` · `axiom:otel-logs:v2` · `axiom:otel-traces:v2` · `axiom:beam-metrics:v2` · `axiom:beam-signals:v2`.
v2 kinds creatable/visible via v2 dataset API only; v1 API keeps legacy behavior. Axiom itself adopts these kinds later — naming is deliberately `axiom:*`.

### C2 — auto-create semantics
- `POST /v1/ingest/{name}` on unknown dataset: requires token with **both `create` and `ingest`** on that name → resolve-or-insert mapping (race-safe on unique org/name) → Metal ensure (idempotent PUT) → forward. Audit row on create.
- Kind at auto-create: OTLP routes → `axiom:otel-logs:v2`/`axiom:otel-traces:v2`; generic ingest → `axiom:events:v2` unless the client sends **`X-Axiom-Dataset-Kind`** (beam sends `axiom:beam-metrics:v2` / `axiom:beam-signals:v2`). Header on existing dataset with mismatched kind → 409.
- Bare `/v1/logs`, `/v1/traces` (no dataset header) → default datasets **`otel-logs`**, **`otel-traces`**, auto-created with their kinds.

### C3 — beam.signals envelope (validated by atom for kind beam-signals)
Required: `_time`, `host.name`, `signal.kind` ∈ {change, condition, inventory, attribution}, `subject.category`, `subject.key`.
Optional envelope: `host.boot_id`, `state`, `prior_state`, `since`, `reason`, `severity`, `evidence.ids`.
Per-category promoted fields are curated (beam owns the list; atom validates envelope only, not promoted fields). Arbitrary shapes ride as JSON strings (`*.before_json` / `*.after_json`). No raw flatten of unbounded objects — ever.

### C4 — beam.metrics row (validated by atom for kind beam-metrics)
Required: `_time` (=window end), `window.start`, `metric.name`, `agg.count`, `agg.sum`, `agg.min`, `agg.max`.
Optional: `metric.unit`, `dims.*` (bounded set: host, gpu.uuid/index, workload.id/kind, adapter.id), `agg.p50/p95/p99`, `bins` (exp-histogram array), `exemplars`. 30s epoch-aligned delta windows; reset = new `window.start`.

### C5 — beam transport
New endpoint scheme `axiom+http(s)://host:port` + bearer token. NDJSON gzip to `/v1/ingest/{dataset}`; `timestamp-field` param maps envelope timestamp; existing idempotency keys preserved. Pipeline output records carry a **target-dataset tag** (`metrics` | `signals` | `otel`); transport routes on the tag. Native-OTel plane: journald/system events emitted as *plain OTel log records* (no beam envelope wrapper) to atom's OTLP logs endpoint.

### C6 — OTLP projection field shape (kind otel-logs/otel-traces + defaults)
Overstream conventions + two fixes: resource attrs `resource.<key>`; record attrs unprefixed; `scope.name`/`scope.version` **retained**; `_time` receipt, `source_time`; spans: hex `trace_id`/`span_id`/`parent_span_id`, `name`, `kind`, `duration_ns`, `status`, `status_message`; logs: `severity` (int), **`severity_text`**, canonical `body`, optional trace correlation.

## Slices

Worktrees: branch `njpatel/bpa-<slice>`, label `bpa/<slice>`, harness omp-rust. Slices 1–4 run in parallel; no slice runs project-wide validation mid-flight (integration phase does).

### 1. `atom-datasets` (atom repo)
Kind column + migration; store/CreateParams/datasetsapi v2 kind support (C1); auto-create on ingest with capability gate, race-safe insert, Metal ensure, audit (C2); bare-OTel default datasets (C2); per-kind ingest admission: otel-* accept only OTLP content types, beam-* validate C3/C4 envelope (reject with clear error), events unchanged; `X-Axiom-Dataset-Kind` handling incl. 409 on mismatch.
**Acceptance**: unit/integration tests — unknown-dataset ingest auto-creates with right kind under create+ingest token and 404s without `create`; bare OTLP creates `otel-logs`; kind mismatch 409; beam-signals row missing `subject.key` rejected; v1 dataset API responses byte-compatible with today.

### 2. `atom-otlp-projection` (atom repo)
Decode-once OTLP → C6 rows → NDJSON to EventDB for otel kinds; replaces raw streaming on those routes (buffer with bounded size); OTLP fixtures → golden projected rows. Coordinate with slice 1 only on the invocation seam (kind lookup selects projection); agree the interface via hub before touching shared ingest handler files.
**Acceptance**: golden tests assert exact C6 field names incl. `scope.name`, `severity_text`; a trace + log fixture round-trips to EventDB-bound NDJSON; non-OTLP kinds untouched (raw streaming preserved for events kind).

### 3. `beam-transport` (beam repo)
`axiom+http(s)` scheme (C5): dataset routing on target tag, bearer auth, gzip NDJSON `/v1/ingest/{dataset}`, `X-Axiom-Dataset-Kind` on first sends, timestamp-field, idempotency preserved; WAL/queue made destination-aware; native-OTel log emission for journald/system events (drop envelope wrapper on that path); config surface (TOML + `BEAM_*`) for base URL/token/dataset names.
**Acceptance**: transport tests against a stub asserting per-dataset routing, headers, kind hints, gzip NDJSON body shape; OTLP logs path emits spec-shaped OTel (no beam envelope); WAL replay lands in correct datasets.

### 4. `beam-signals` (beam repo)
beam-schema: `condition_event` kind; C3 envelope emission — normalize adapter_status/GPU cycle/sensor lifecycle → condition rows (anomaly=enter-abnormal, recovery=return-to-normal); inventory as row-per-entity + changes in envelope shape with promoted fields + JSON residue; pipeline metric windows serialized as C4 rows (bins on); target-dataset tagging (C5) on all outputs.
**Acceptance**: schema/unit tests — a package-diff produces a C3-valid change row with `package.version.before/after`; adapter failure→running produces condition pair; metric window serializes to C4 with bins; every emitted record carries a target tag.

### 5. Integration + e2e (orchestrator — me, this repo)
Merge order 3→4 (beam) and 1→2 (atom) with conflict resolution; stand up the stack; run the acceptance scenario; capture evidence (APL queries + results, Grafana screenshots) into `beam-prism-atom/evidence/`.

## Pre-flight checks (before creating worktrees)

- **P1 — Metal availability**: confirm how to run Metal EventDB locally (atom `make demo-up` stand-in vs real metal build from axiomhq/axiom). E2E depends on it; atom compose expects it at `:18085`.
- **P2 — Grafana Axiom plugin compat**: confirm the plugin's API calls (APL tabular endpoint, dataset listing) against what atom serves; note any gaps as slice-1 addenda.

## Risks

- Two agents per repo → shared-file merge risk; mitigated by contract pinning + hub coordination + orchestrator-owned merges (slice 5).
- beam-* envelope validation (slice 1) depends on C3/C4 stability — contracts frozen above; changes require Neil.
- Grafana plugin may require endpoints atom lacks (P2 surfaces this before work starts).
- Metal stand-in may not exercise real APL aggregation (P1 decides simulator vs real).

## Post-execution status (2026-08-28)

DONE. All four slices delivered by parallel agents, merged on `njpatel/bpa-integration` in both repos, six-step acceptance scenario passed against a real EventDB (axiom `cmd/db`) with Grafana+Axiom-plugin as the read surface. Evidence: `evidence/e2e-acceptance.md` + screenshot.

**Contract amendment discovered in integration (C3/C4/C6 wire form):** rows ship as **nested JSON**; EventDB flattens nested objects into unescaped dot-path field names (`['signal.kind']`), while literal-dot flat keys become escaped fields (`['signal\.kind']`) that poison APL. Flat dotted top-level keys are rejected by beam-* admission. Stored/queried field names are unchanged — only the wire encoding nests.

Three integration-phase fixes beyond the slices (all committed on integration branches): atom APL dataset-name parser treating field refs as dataset refs; beam standalone-node startup deadlock on initial inventory emission; timestamp-field `_time` alignment. Follow-ups listed in the evidence doc.

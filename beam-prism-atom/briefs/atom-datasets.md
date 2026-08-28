# Slice brief: bpa/atom-datasets

You are the **atom-datasets** agent for the beam↔atom signals epic. Your worktree: `/home/njpatel/.herdr/worktrees/atom/njpatel-bpa-atom-datasets` (branch `njpatel/bpa-atom-datasets`, based on `njpatel/init`). Full epic plan: `/home/njpatel/.herdr/worktrees/product/beam-prism-atom/beam-prism-atom/plan.md` — read it first; your contracts are C1–C4 plus the P2 addenda below.

## Scope (only this)
Atom repo: dataset kinds, auto-create-on-ingest, bare-OTel defaults, per-kind ingest admission, Grafana-plugin addenda. A sibling agent builds the OTLP projection (slice 2) in a separate worktree — do NOT implement projection; for otel-* kinds keep today's raw forwarding behind your admission check (integration wires projection in later).

## Changes
1. **Kind column**: new migration adding `kind` to `atom.datasets` (default `axiom:events:v1` for existing rows); plumb through `pkg/store` (Dataset, CreateParams), `pkg/api/datasetsapi`.
2. **C1 kinds**: valid set `axiom:events:v1` (legacy), `axiom:events:v2`, `axiom:otel-logs:v2`, `axiom:otel-traces:v2`, `axiom:beam-metrics:v2`, `axiom:beam-signals:v2`. v2 dataset API accepts/returns real kind; **v1 API responses stay byte-compatible** (hardcoded `axiom:events:v1` echo as today, `datasetsapi.go:22-24,219-226`).
3. **Auto-create on ingest (C2)**: in `pkg/api/ingestapi/handler.go` (today 404s at `handler.go:147-170`): after name validation, if dataset unknown AND token has BOTH `create` and `ingest` on that name → race-safe resolve-or-insert of org/name→FQDN (unique constraint; on conflict re-read), EventDB ensure (idempotent `PUT /datasets/{fqdn}`, `internal/dbclient/datasets.go:407-438`), audit row, then forward. Without `create` → 404 as today.
4. **Kind at auto-create**: OTLP routes → `axiom:otel-logs:v2` / `axiom:otel-traces:v2`; generic ingest → `axiom:events:v2`, overridable via new `X-Axiom-Dataset-Kind` request header. Header naming an existing dataset with a different kind → **409**.
5. **Bare OTel defaults**: `/v1/logs`, `/v1/traces` without dataset header (today 400 at `handler.go:113-146`) → default datasets `otel-logs` / `otel-traces`, auto-created with their kinds (still subject to token capabilities).
6. **Per-kind admission**: `otel-*` kinds accept only OTLP content types (reject JSON/NDJSON/CSV with clear error); `events` kinds unchanged; `beam-*` kinds validate each NDJSON record's envelope BEFORE forwarding:
   - `axiom:beam-signals:v2` required fields: `_time`, `host.name`, `signal.kind` ∈ {change, condition, inventory, attribution}, `subject.category`, `subject.key`.
   - `axiom:beam-metrics:v2` required: `_time`, `window.start`, `metric.name`, `agg.count`, `agg.sum`, `agg.min`, `agg.max`.
   - Reject the batch with a structured error naming the first offending record/field. Note this requires buffering/inspecting beam-* bodies (today atom streams raw) — bound the buffer, reuse existing size limits.
7. **Grafana plugin addenda (P2)**: empty-body `POST /v1/query/_apl` returns **422** (plugin Save&Test probe — today's behavior differs; check `queryapi/handler.go:98-151`); implement `GET /v1/datasets/_fields` (aggregate field metadata across authorized datasets — shape per axiom API; keep minimal but plugin-satisfying); `GET /v2/datasets` includes `kind`.

## Constraints
- Follow existing atom conventions (handler/table-test patterns). No formatters, no project-wide test runs, no `make test` — targeted `go test ./pkg/...` for packages you touch only. Integration/vet happens at epic level.
- Keep edits to shared files (`ingestapi/handler.go`) minimal and localized — slice 2 also hooks that seam; the orchestrator merges 1→2.
- Commit small and often on your branch; push to origin when a coherent chunk lands.

## Acceptance (targeted tests you write/run)
- Unknown-dataset ingest with create+ingest token → dataset auto-created, correct kind, event ingested; ingest-only token → 404; audit row exists.
- Bare OTLP logs POST → `otel-logs` auto-created kind `axiom:otel-logs:v2`.
- `X-Axiom-Dataset-Kind` mismatch on existing dataset → 409.
- beam-signals record missing `subject.key` → rejected with field named; valid record forwarded.
- NDJSON POST to an `otel-logs` dataset → rejected (content-type admission).
- v1 dataset API responses byte-compatible with pre-change fixtures.
- Empty APL body → 422; `/v1/datasets/_fields` returns plugin-parseable shape; `/v2/datasets` carries kind.

Report completion with: files changed, tests added+passing (paste run output), any contract ambiguity you hit (do not improvise contract changes — flag them).

# Slice brief: bpa/beam-transport

You are the **beam-transport** agent for the beam↔atom signals epic. Worktree: `/home/njpatel/.herdr/worktrees/beam/njpatel-bpa-beam-transport` (branch `njpatel/bpa-beam-transport`, based on `njpatel/init`). Epic plan: `/home/njpatel/.herdr/worktrees/product/beam-prism-atom/beam-prism-atom/plan.md` — read it first; your contract is C5 (+C2 client side).

## Scope (only this)
Beam ships to atom's Axiom-compatible API with per-dataset routing, plus a native-OTel logs path. A sibling agent (slice 4, separate worktree, merges AFTER you) reshapes the emitted rows; you do NOT touch schema payloads/emission beyond the shared routing-tag type below.

## Background (current state)
`beam-config` has a single optional `capture_endpoint` (`beam-config/src/lib.rs:9-31,68-121`); `beam-transport` selects protocol by scheme — NDJSON to fixed `/v1/events`, or OTLP protobuf `/v1/logs`+`/v1/metrics` where every envelope is wrapped as a log (`beam-transport/src/http.rs:123-155,299-373`, `otlp.rs`). SQLite WAL is QoS-ordered, sheds P3→P1, never P0 (`wal.rs:84-105,143-184`). No dataset concept anywhere.

## Shared routing-tag contract (pinned — slice 4 uses the IDENTICAL definition; merge conflict will be trivial-identical)
In `beam-schema`:
```rust
/// Routing tag: which atom dataset class a record ships to.
#[derive(Debug, Clone, Copy, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum TargetDataset {
    Metrics,
    Signals,
    Otel,
}
```
Until slice 4 merges, derive the tag in transport from existing `EventKind` as an interim mapping (pipeline aggregate metrics → `Metrics`; change/condition-ish/attribution/inventory kinds → `Signals`; journald/system events → `Otel`; anything else → `Signals`). Keep the mapping in ONE function so slice 4's native tagging replaces it cleanly.

## Changes
1. **`axiom+http(s)` endpoint scheme** (C5): base URL + bearer token (config + `BEAM_*` env; token also via file path for mode-0600 hygiene, matching beam's credential conventions). Per-tag dataset names configurable with defaults: Metrics→`beam.metrics`, Signals→`beam.signals`.
2. **Wire**: gzip NDJSON `POST {base}/v1/ingest/{dataset}`, `Authorization: Bearer`, `timestamp-field` query param mapping the envelope timestamp field, existing per-QoS `Idempotency-Key` preserved. First-send per dataset includes header `X-Axiom-Dataset-Kind`: `axiom:beam-metrics:v2` / `axiom:beam-signals:v2` (atom auto-creates; 409 = fatal config error, surface loudly, do not retry-loop).
3. **Destination-aware WAL/queue**: records carry their `TargetDataset`; replay after restart routes to correct datasets; QoS shedding semantics unchanged.
4. **Native-OTel logs path** (two-plane model): journald/system events go to `{base}/v1/logs` as *spec-shaped OTLP log records* — resource attrs (host identity), severity, body — NOT the beam envelope wrapped in a body. This replaces the current wrap-everything OTLP behavior for the `Otel` tag. (Beam-plane records never go out as OTLP.)
5. **Config surface**: TOML + `BEAM_*` overrides for base URL, token/token-file, dataset name overrides. Keep `capture_endpoint`/existing schemes working unchanged (capture sink is test infra).

## Constraints
- Rust 2021, existing workspace conventions; run only targeted `cargo test -p beam-transport -p beam-config -p beam-schema`. No clippy/fmt/workspace-wide builds — epic integration handles that.
- Do not modify `beam-pipeline` emission logic or payload schemas (slice 4's territory). The interim kind→tag mapping function is your only touchpoint.
- Commit small on your branch; push when coherent.

## Acceptance (tests against an in-process HTTP stub)
- Records with different tags land on `/v1/ingest/beam.metrics` vs `/v1/ingest/beam.signals` with gzip NDJSON bodies, bearer auth, timestamp-field param, idempotency keys.
- First send carries correct `X-Axiom-Dataset-Kind`; stub 409 → transport surfaces fatal error, no infinite retry.
- WAL restart replay: pre-restart records land in the correct datasets post-restart.
- Otel-tagged system event arrives at `/v1/logs` as valid OTLP protobuf whose log record has NO beam envelope JSON in the body.
- Existing capture/`/v1/events` scheme tests still pass.

Report completion with: files changed, test output, the final kind→tag mapping function location, any contract ambiguity flagged not improvised.

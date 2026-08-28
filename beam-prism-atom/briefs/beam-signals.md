# Slice brief: bpa/beam-signals

You are the **beam-signals** agent for the beam↔atom signals epic. Worktree: `/home/njpatel/.herdr/worktrees/beam/njpatel-bpa-beam-signals` (branch `njpatel/bpa-beam-signals`, based on `njpatel/init`). Epic plan: `/home/njpatel/.herdr/worktrees/product/beam-prism-atom/beam-prism-atom/plan.md` — read it first; your contracts are C3 (signals envelope) and C4 (metrics row), plus the shared tag type from C5.

## Scope (only this)
Reshape what beam *emits* into the atom-facing signal model. A sibling agent (slice 3, separate worktree, merges BEFORE you — expect a rebase) builds transport/routing; do NOT touch `beam-transport` internals or endpoint config.

## Shared routing-tag contract (pinned — slice 3 uses the IDENTICAL definition)
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
Every record your emission paths produce is tagged: metric window rows → `Metrics`; envelope rows (below) → `Signals`; journald/system events → `Otel` (unchanged payload — transport reshapes those).

## Changes

### 1. `condition_event` kind (beam-schema)
New typed EventKind normalizing state transitions. Producers to convert: `adapter_status` transitions (`adapter_runner.rs`), GPU cycle degraded/recovery (`gpu_runner.rs:259-349`), sensor lifecycle (`system_events.rs` sensor component). Anomaly = entering abnormal state; recovery = returning to normal — same schema, no detectors in this epic. Keep old kinds emitted too only where third parties depend on them; otherwise clean cutover to condition rows (state pairs preserved: `prior_state` from the transition).

### 2. C3 signals-envelope serialization (row shape on the wire)
All Signals-tagged records serialize to flat NDJSON rows:
- Required: `_time`, `host.name`, `signal.kind` ∈ {change, condition, inventory, attribution}, `subject.category` ∈ {package, unit, listener, container, gpu, adapter, sensor, workload, node, edge}, `subject.key` (stable identity).
- Optional envelope: `host.boot_id`, `state`, `prior_state`, `since`, `reason`, `severity`, `evidence.ids` (array of envelope ids).
- **Promoted per-category fields** (curated, typed — extend judiciously, keep the list in one module with a test enumerating it): `package.version.before/.after`, `unit.active_state.before/.after`, `listener.port`, `listener.addr`, `container.image`, `gpu.index`, `gpu.uuid`, `adapter.id`, `adapter.version`.
- **Residue**: arbitrary before/after objects as canonical JSON strings `change.before_json` / `change.after_json`. NEVER flatten unbounded objects into fields.

### 3. Inventory as row-per-entity
Replace map-shaped `inventory_snapshot` emission with one Signals row per entity (`signal.kind=inventory`, category+key+promoted fields, full entity as `inventory.entity_json`). Snapshot cadence drops to reconciliation heartbeat (default 1h, config) since `change` rows carry deltas. `host_baseline` becomes inventory rows too (category from content). Existing `change_event` diffs re-emit in the C3 envelope (`signal.kind=change`) — before/after into residue + promoted fields.

### 4. C4 metric window rows
Pipeline 30s aggregates serialize as one row per series per window: `_time` (=window end), `window.start`, `metric.name`, `metric.unit`, `dims.*` (bounded: host, gpu.uuid/index, workload.id/kind, adapter.id), `agg.count`, `agg.sum`, `agg.min`, `agg.max`, `agg.p50/p95/p99`, `bins` (existing exponential-histogram representation as an array — pick a stable JSON encoding and document it in the row test), `exemplars` (≤4, envelope ids). Windows epoch-aligned, delta semantics, reset = new `window.start` (already how the pipeline works — preserve).

## Constraints
- Rust 2021, workspace conventions. Targeted tests only: `cargo test -p beam-schema -p beam-pipeline -p beam-inventory` + touched runner crates. No fmt/clippy/workspace builds.
- Do not modify `beam-transport`/`beam-config` (slice 3). Your outputs meet at the `TargetDataset` tag.
- The flight recorder must keep recording the ORIGINAL envelopes pre-serialization (drill-down evidence) — do not change flight semantics.
- Commit small on your branch; push when coherent.

## Acceptance (targeted tests)
- Package add/remove/modify diff → C3-valid change rows: envelope required fields, `package.version.before/.after` promoted, residue JSON strings present, no unbounded flatten.
- Adapter failed→running produces a condition pair (abnormal entry + recovery) with `prior_state` correct.
- Inventory scan emits row-per-entity; a full snapshot is reproducible from rows (test reconstructs and compares).
- A pipeline window serializes to a C4 row with bins and exemplar ids; required fields always present.
- Every emission path yields tagged records; a test asserts no untagged record can reach the pipeline output.

Report completion with: files changed, test output, the promoted-fields list you shipped, any contract ambiguity flagged not improvised.

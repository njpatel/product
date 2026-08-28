# Slice brief: bpa/atom-otlp-projection

You are the **atom-otlp-projection** agent for the beam↔atom signals epic. Worktree: `/home/njpatel/.herdr/worktrees/atom/njpatel-bpa-atom-otlp-projection` (branch `njpatel/bpa-atom-otlp-projection`, based on `njpatel/init`). Epic plan: `/home/njpatel/.herdr/worktrees/product/beam-prism-atom/beam-prism-atom/plan.md` — read it first; your contract is C6.

## Scope (only this)
Replace atom's raw OTLP forwarding with decode-once → typed projected rows → NDJSON to EventDB. A sibling agent (slice 1, separate worktree) is adding dataset kinds + admission; you do NOT implement kinds, auto-create, or validation. Build the projection as a **new package** with a narrow entry point; keep `ingestapi/handler.go` edits minimal (a single call-site hook) — the orchestrator merges slice 1 first, then yours.

## Background
Today atom streams OTLP bodies byte-identical to Metal (`pkg/api/ingestapi/handler.go:172-181`, proven by `handler_test.go:374-429`); Metal's axiom-derived parser produces hard-to-query dotted sprawl. The reference implementation is overstream (Neil's testing ground): read `/home/njpatel/.herdr/worktrees/overstream/worker-research/internal/gateway/decode.go` (OTLP decode; note it DROPS scope — you must not) and `internal/axiom/ingest.go:149-218` (row projection). You are porting the *conventions* to atom in Go, not vendoring the code.

## Output row shape (C6 — exact field names)
Common: `_time` (receipt time), `source_time` (producer timestamp when present), resource attrs as `resource.<key>`, record attrs unprefixed, `scope.name`, `scope.version` (**retained — fix vs overstream prototype**).
Spans: `trace_id`, `span_id`, `parent_span_id` (lowercase hex), `name`, `kind` (int enum), `duration_ns` (end−start), `status` (int code), `status_message` (when set).
Logs: `severity` (int SeverityNumber), `severity_text` (**added — fix vs overstream prototype**), `body` (canonical: string/scalar directly; compound bodies as canonical JSON string — do NOT emit opaque binary like overstream does), optional `trace_id`/`span_id` when nonzero.
AnyValue mapping: scalars typed; arrays/kvlists → canonical JSON string. Attribute keys used as-is (dot-paths allowed, ≤200 bytes per EventDB limit — truncate-and-flag or drop-and-count, your call, but deterministic and tested).

## Changes
1. New package (e.g. `pkg/otelproject`): decode OTLP protobuf AND JSON (both content types atom accepts) for logs + traces → `[]map[string]any` rows or an io streaming equivalent; then NDJSON-encode for EventDB ingest with `timestamp-field=_time`.
2. Bounded buffering: OTLP bodies must now be read fully before forward — enforce a size limit (reuse atom's existing limit conventions; reject over-limit with clear error).
3. Hook: one call-site in the OTLP ingest path that routes OTLP content types through projection instead of raw forward. Gate it so non-OTLP paths are untouched. (After merge with slice 1 this gate becomes a kind check; design the hook so that's a one-line change.)
4. Golden tests: OTLP log + trace fixtures (protobuf and JSON) → assert exact projected field names/values per C6, including `scope.name`, `severity_text`, hex IDs, `duration_ns`, canonical compound body.

## Constraints
- Existing atom conventions; table tests; no new deps beyond what OTLP decoding needs (check what `internal/dbclient` / vendored code already provides before adding a proto module).
- No formatters, no project-wide tests/vet — targeted `go test` on your packages only.
- Commit small on your branch; push when coherent.

## Acceptance
- Golden tests pass: fixture OTLP logs/traces (pb + JSON) produce exactly the C6 rows; byte-stable NDJSON output.
- Raw-forward path for non-OTLP content types provably untouched (existing tests still green).
- Over-limit OTLP body rejected with structured error.

Report completion with: files changed, test output, the exact hook signature you exposed (slice 1/orchestrator needs it), any contract ambiguity flagged not improvised.

# Adaptive loop — leader, enrollment, and capability evolution on the live fleet

Run: 2026-08-28 (fleet from `fleet-confidence.md` still live). Goal: prove beam *adapts* — control plane leader, fleet enrollment, and the full capability request→approval→signed delivery→running loop.

## Leader

`beam serve control` on adipurush, `0.0.0.0:19419` (tailnet-protected), SQLite state, mode-0600 credential registry with per-node base64 bearer tokens (adipurush-bpa / adiyogi / bpa-cloud-1, tenant `axiom`, group `bpa-fleet`). Fleet profile set `full` via `beam control profile set`.

All three nodes re-pointed from `--standalone-profile` to `--control-url` + `--control-token-file`:
- adipurush-bpa → `ws://127.0.0.1:19419`
- adiyogi → `ws://100.127.175.40:19419` (tailnet)
- bpa-cloud-1 → `ws://127.0.0.1:19419` over the reverse SSH tunnel (second `-R` forward)

`beam control nodes`: all three `connected: true`, `effective_matches_desired: true`.

## Beam bug found & fixed (committed to `njpatel/bpa-integration`)

**Control WS handshake failed over WAN/tunnel** (`Interrupted handshake (WouldBlock)`): the 250 ms operational read timeout — which doubles as the session loop tick — was set *before* the handshake, so any path slower than 250 ms RTT aborted enrollment. Fix: handshake under `CONTROL_CONNECT_TIMEOUT` (5 s), apply the session timeout after upgrade. Cloud enrollment succeeded immediately after.

## The evolution loop (postgres capability)

Machinery: built `postgres-ready` WASI **component** (core wasm must be componentized via xtask — see lesson 2), Ed25519 keypair via `beam adapter keygen`, manifest signed + verified, signer pubkey distributed to nodes as `<signer_key_id>.pub` (0600, in 0700 dirs), nodes restarted with `--adapter-dir`/`--adapter-trust-key`.

Sequence, all observable in the control audit trail and atom:

1. `apt install postgresql` on adiyogi → **node filed `needs_build` capability request within 20 s** (evidence: dpkg + processes + listeners + units).
2. `request attach --manifest --artifact --risk low` → `awaiting_approval` (package NOT deployed before approval — verified).
3. `request approve` → `approved` → control delivered the signed package; node verified the Ed25519 signature and staged atomically.
4. `request list` → **`deployed`, `coverage: covered`**.
5. atom `beam.signals`: `postgres-ready` condition `running` — "version 1.0.0 active at generation 1"; `beam.metrics`: **`beam.adapter.postgres_ready.value` windows flowing from adiyogi** — the delivered adapter probes postgres every 2 s and its telemetry rides the full pipeline.

The earlier failed attempt is preserved honestly: bpa-cloud-1 condition `failed` — "adapter artifact is not a WebAssembly component; core modules are not supported".

## Semantics learned (valuable for docs/upstream)

1. **Whitelist implies coverage**: whitelisting an adapter whose id covers a capability suppresses new requests (`capability_is_covered`). The postgres request only fires with `postgres-ready` *off* the whitelist — matching the evolution test's own choreography. Operator-attached packages deliver via request approval regardless of whitelist.
2. **Adapters must be WASI components**, not core wasm modules — componentize via `xtask build-test-adapters`.
3. **`suspended` and `rejected` are terminal**; nodes do not re-file a suspended request_key (operator retirement is respected). Recovery from a bad attach = fresh request (new node here; upstream may want an explicit resubmit path).
4. Strict mode enforcement everywhere: trust dirs 0700, keys 0600 named `<signer_key_id>.pub`, adapter dir 0700, `HOME` required under systemd unless attribution paths are explicit. Good security posture; deploy scripts must comply.

## Vultr status

API flaps: intermittent 200/401/500 on reads, consistent 401 "Invalid API token" on writes after the IP allowlist edit — likely the key was regenerated or is read-scoped. Needs the current key value re-checked in the Vultr portal.

# Fleet confidence run — real servers, real data, real dashboards

Run: 2026-08-28. Goal: build confidence that the beam→atom signal model shows what we expect on a real multi-host fleet before changing upstream.

## Fleet

| Host | Kind | Sensor (eBPF) | Path to atom | Deploy |
|---|---|---|---|---|
| adipurush | local workstation | no (unprivileged) | localhost | hub-managed process |
| adiyogi | lab host, 64c, 2×4090 | **yes** (root systemd) | tailnet → `100.127.175.40:55080` | systemd units via sudo |
| bpa-cloud-1 | Latitude.sh bare metal `c3-small-x86` (SAN3, Xeon E-2386G, 32GB, `45.250.252.23`, hourly) | **yes** | reverse SSH tunnel (hub-managed, auto-restart) → atom stays Tailscale-private | systemd units via sudo |

Vultr blocked: API key IP-allowlist rejects our shared egress `86.97.92.54` — needs Neil to allowlist in the Vultr portal. Latitude.sh worked immediately (project `axiom-test`, server `sv_BDXM5EkX70rpk`).

Per-host `xaat-` tokens (`bpa-fleet-*`), mode-0600 token files. First deploy hiccup: beam requires `HOME` or explicit attribution paths under systemd — fixed with explicit `--attribution-*` paths.

## Dashboards (provisioned as code, `dashboards/`)

- **Beam Fleet — Overview** (`fleet-overview-dashboard.webp`): 3 hosts reporting; metric windows/min per host; available-memory p95 per host (adiyogi ~192GiB vs adipurush ~56GiB vs cloud); process count per host (adipurush ~1700, adiyogi ~1000, cloud ~250); GPU memory (adiyogi real workloads); OTel log volume.
- **Beam Fleet — Changes & Conditions** (`fleet-signals-dashboard.webp`): change counts by host/category; live condition-transition table telling the full story — `beam-sensor loaded eBPF → running`, `beam-sensor/client connected`, `nvidia_smi degraded` on the GPU-less cloud host (correct!); live change feed.

## Verification matrix (APL, all against atom)

| Expectation | Result |
|---|---|
| 3 hosts reporting metrics in 5m | ✓ `dcount(dims.host) = 3` |
| Package install visible w/ versions | ✓ `adiyogi sl None→5.02-1`, `bpa-cloud-1 cowsay None→3.03+dfsg2-8` **+ its dependency libtext-charwidth-perl** (transitive deps caught) |
| Package remove visible | ✓ `adiyogi sl 5.02-1→None` |
| Unit stop/start w/ states | ✓ `unattended-upgrades active→inactive` then `inactive→active` (promoted `unit.active_state.*`). Lesson: `systemctl restart` is too fast for the 10s scan without the sensor-trigger path — dwell needed |
| Listener open on each host | ✓ all three (`:55666` cloud, `:55667` adiyogi, `:55999` adipurush) |
| Container add/remove | ✓ adipurush (4 change rows) |
| Sensor lifecycle conditions | ✓ `loaded eBPF → running`, `client connected` on both sensor hosts |
| GPU degraded on GPU-less host | ✓ cloud `nvidia_smi_procfs degraded` — honest signal, not noise |
| Inventory per host | ✓ 2173/1265/332 packages; units/listeners/containers proportional |
| OTel logs attributed per host | ✓ adiyogi 24 / adipurush 18 / cloud 12 |
| **Traces plane (first exercise)** | ✓ bare `POST /v1/traces` auto-created `otel-traces` (kind v2); parent/child spans, hex ids, `duration_ns`, `db.system` attr, all unescaped |
| Real GPU workload metrics | ✓ `beam.gpu.process.used_memory` p95 from adiyogi's live processes |
| Sensor kernel-plane health | ✓ `beam.host.sensor.{delta,cumulative,per_cpu}` counters from both sensor hosts |

## Observations / refinements queued

1. Kernel exec/exit activity is visible via sensor counters but not as first-class `beam.metrics` series — pipeline aggregate event-counts ride the self-telemetry OTel plane. Decide if exec-rate deserves promotion to a metric series.
2. "Degraded subjects" panel needs a per-subject healthy-state vocabulary (`connected` is healthy for `beam-sensor/client` but != `running`).
3. `systemctl restart` invisible without sensor triggers at 10s scan — on sensor hosts the trigger path should catch it; verify trigger-driven rescan latency as a follow-up.
4. Latitude server bills hourly — teardown decision is Neil's (`sv_BDXM5EkX70rpk`).

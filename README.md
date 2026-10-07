# Ceph by HTTP (Prometheus) — Zabbix Template

Monitors a Ceph cluster through its built-in **mgr Prometheus exporter** — no
`zabbix-agent2` Ceph plugin, no deprecated `restful`/`zabbix` mgr modules
required. Works with modern Ceph releases (tested on 20.2.x / Tentacle,
where the old integration paths are removed).

## What it monitors

- **Cluster health** — overall status, and every individual Ceph health
  check (`BLUESTORE_SLOW_OP_ALERT`, `PG_NOT_DEEP_SCRUBBED`, etc.) via
  auto-discovery — new check types show up automatically, nothing to edit.
- **OSDs** — auto-discovered. Per-OSD commit/apply latency, up/in status,
  slow-ops flag, plus cluster-wide up/in/out/down counts.
- **Monitors** — auto-discovered. Quorum status and slow-ops flag per mon.
- **Placement Groups** — total/active/clean/degraded/undersized/remapped/
  backfilling/backfill_wait/recovering/inconsistent/down/incomplete, plus a
  calculated "percent not clean" item.
- **Pool** (one RBD pool, configurable) — percent used, stored bytes, max
  available, object count, read/write bandwidth and IOPS.
- **Cluster capacity** — total/used bytes and percent.
- **Misc** — active mgr count, misplaced/unfound/degraded object counts,
  recovery throughput, all 9 OSD maintenance flags (`noout`, `nobackfill`,
  etc. — alerts if one is left set).

## Requirements

- Ceph mgr Prometheus module enabled on the cluster:
  ```bash
  ceph mgr module enable prometheus
  ```
- The Zabbix server (or proxy) must be able to reach
  `http://<active-mgr-ip>:9283/metrics` over the network.
- Zabbix 6.4+ (uses HTTP agent items, dependent items, Prometheus
  pattern/to-JSON preprocessing, and LLD with Prometheus discovery).

## Import

1. **Data collection → Templates → Import**, select
   `zbx_export_ceph_by_http_prometheus.yaml`, click **Import**.
2. Create (or pick) a host to represent your Ceph cluster, e.g. `MyCephCluster`.
3. Link the template **Ceph by HTTP Prometheus** to that host.
4. On the host, override at least `{$CEPH.API.URL}` (see macro table below).
5. Wait one discovery interval (or run the discovery rules manually) — OSD
   and Monitor items/triggers will appear automatically.

## Macros (override per host as needed)

| Macro | Default | Purpose |
|---|---|---|
| `{$CEPH.API.URL}` | `http://localhost:9283/metrics` | **Must override.** Point this at your active mgr's Prometheus endpoint. If mgr failover is a concern, point it at a VIP/load balancer in front of the mgr nodes, or monitor each mgr node separately. |
| `{$CEPH.RBD.POOL.ID}` | `6` | **Must override** unless it happens to match. Find your pool's numeric ID with `ceph osd pool ls detail`. |
| `{$CEPH.OSD.LATENCY.WARN}` | `500` | OSD commit-latency threshold (ms, sustained 5 min) for the "latency critical" trigger. |
| `{$CEPH.POOL.USED.WARN}` | `80` | Pool percent-used warning threshold. |
| `{$CEPH.POOL.USED.CRIT}` | `90` | Pool percent-used critical threshold. |
| `{$CEPH.SLOWOPS.WARN}` | `10` | Cluster-wide slow-ops count warning threshold. |
| `{$CEPH.SLOWOPS.CRIT}` | `50` | Cluster-wide slow-ops count critical threshold. |
| `{$CEPH.PG.NOTCLEAN.WARN}` | `20` | Percent of PGs not "clean" — warning threshold. |

## Finding your pool ID

```bash
ceph osd pool ls detail
# pool 6 'rbd-pool' replicated size 3 ...
#      ^ this number is {$CEPH.RBD.POOL.ID}
```

If you need to monitor more than one pool, duplicate the `rbd-pool *` items
(search for `pool_id="{$CEPH.RBD.POOL.ID}"` in the template) and point the
copies at a second macro, or convert pools to an LLD discovery rule the same
way OSDs/Monitors are handled here (pattern: `ceph_pool_percent_used`,
LLD macro `{#POOL_ID}` = `$.labels['pool_id']`).

## Dashboard

A ready-made dashboard is **deliberately not bundled** in this YAML. Template
dashboard widgets use an internal field schema that's easy to get subtly
wrong by hand, and a mistake there risks breaking the *entire* import file —
including the items/triggers that are already tested and working. Build the
dashboard once in the UI instead (takes a few minutes, and it's then part of
the template going forward):

1. **Data collection → Templates → Ceph by HTTP Prometheus → Dashboards tab
   → Create dashboard**.
2. Add widgets (Graph, Gauge, Item value, etc.) and pick items by **key**
   (no host is selected — template dashboards resolve to whichever host the
   template is linked to).
3. Suggested layout:
   - Item value / Gauge row: `ceph.health.status`, `ceph.osd.count_down`,
     `ceph.pool.used[rbd]`, `ceph.slow_ops`.
   - Graph: all `ceph.osd.commit[*]` latency items together.
   - Graph: `ceph.pg.*` state items together.
4. Once it looks right, re-export the template — the dashboard will be
   included automatically in that export.

## Notes on design choices

- **OSDs and Monitors use LLD (auto-discovery)**, not hardcoded per-daemon
  items — so this template works unmodified on a cluster with 3 OSDs or 300.
- **Aggregated PG/flag counts use the Prometheus pattern "sum" result
  processing** (not PromQL `sum(...)` syntax — Zabbix's own query language
  differs from PromQL here) to collapse per-pool metrics into cluster-wide
  totals.
- Counters (`ceph_pool_rd_bytes`, `wr_bytes`, `rd`, `wr`) are cumulative, so
  the bandwidth/IOPS items use an extra **"Change per second"**
  preprocessing step. `ceph_pool_recovering_bytes_per_sec` is already a
  rate and does *not* get that step.

## License

MIT — use, modify, and redistribute freely.

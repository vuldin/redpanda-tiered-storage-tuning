# 4. Tuning guide

## The loop

This is the actual procedure - the property sections below are what to slot
into step 3, not a substitute for running it.

1. **Baseline.** Run `02-benchmark-write-path.md` and `03-benchmark-read-path.md`
   (including the read-path concurrency sweep) once, unmodified, and record
   the numbers - achieved MB/s for write-path, and the throughput-vs-concurrency
   curve for read-path. This is iteration 0. Do not skip recording it: without
   a written baseline, "did that help?" three properties later is a guess.
2. **Pick one lever.** Use the ordering below (connection concurrency, then
   segment sizing, then read-side cache/readers) and the symptom-to-property
   mapping in the decision tree at the bottom. Change exactly one property.
3. **Re-run the same benchmark.** Same workload file, same driver file, same
   duration you used for the baseline - only the one property changed.
   Restart brokers first if the property needs it (check each property's
   "Requires restart" note).
4. **Compare against the previous iteration**, not just the original
   baseline - gains can compound or cancel across changes.
   - Meaningfully better -> keep the change, this number is the new
     baseline, go back to step 2 for the next lever.
   - No change, or worse -> revert the property, go back to step 2 and try
     the next lever instead.
5. **Stop when either is true:**
   - The last one or two changes bought you less improvement than you'd
     act on (diminishing returns) - the current configuration is your
     production candidate.
   - Achieved throughput has converged with the appliance/network ceiling
     from `02-`'s "Reading the result" section, or the read-path
     concurrency sweep has flattened - you've hit a hardware limit, and no
     further Redpanda property is going to move it.

A simple log (one row per iteration) is enough to keep this honest:

| Iteration | Property changed | Old -> new value | Write-path MB/s | Read-path MB/s @ concurrency | Kept? |
|---|---|---|---|---|---|
| 0 (baseline) | - | - | | | - |
| 1 | `cloud_storage_max_connections` | 20 -> 40 | | | |

Work through the levers in this order: connection concurrency first, then
segment sizing, then the read-side cache and reader limits - these
properties interact, and it's easy to mask one bottleneck by fixing
another out of order.

All properties below are cluster-wide config, set with
`rpk cluster config set <property> <value>`. Check the "restart" column
before you plan a change.

## Connection concurrency - the first lever to pull

### `cloud_storage_max_connections`

Maximum simultaneous object storage connections **per shard (CPU core)**,
shared by upload and download - a connection in use for an upload cannot
serve a download at the same moment. Default **20**. Requires restart.

This is the property public-cloud defaults are built around: 20
connections/core is a reasonable ceiling against a rate-limited public
endpoint. Against a local appliance with single-digit-millisecond
round-trip latency (see `01-connect-and-validate.md`'s self-test numbers),
20 connections/core can leave real headroom on the table - each connection
completes its work faster, so the same connection count moves less data
per second than it would against a higher-latency endpoint.

If your write-path or read-path benchmark plateaus while
`vectorized_ntp_archiver_pending` (write side) or consumer throughput
(read side) suggests Redpanda itself is the limit - not the network or the
appliance's own front end - raise this incrementally and re-run the
benchmark. Watch appliance-side CPU/connection metrics if your storage team
can share them; there is a point past which more Redpanda-side concurrency
just queues up in front of an appliance that's already at its own limit,
and that's a hardware ceiling, not a tuning one.

### `cloud_storage_max_concurrent_hydrations_per_shard`

Maximum concurrent segment hydrations (downloads) per core. Default
**unset**, which resolves to `cloud_storage_max_connections / 2` - i.e. by
default, hydration is capped at half of whatever `cloud_storage_max_connections`
allows. No restart needed.

If you raise `cloud_storage_max_connections` for the write path but the
read-path benchmark doesn't move, check whether this property is pinned to
an old explicit value rather than tracking the new `max_connections/2`
default - it can quietly become the binding constraint on the read side
even after the shared connection pool grows.

### Endpoint-specific connectivity

If self-test latencies were higher than expected, or the write/read
benchmarks show high retry rates rather than a clean plateau, revisit the
connectivity properties in `01-connect-and-validate.md` first -
`cloud_storage_url_style` mismatches, TLS termination assumptions, or a
load balancer in front of the appliance can all look like a Redpanda
tuning problem when they aren't one.

## Segment sizing - the second lever

### `segment.bytes` (topic) / `log_segment_size` (cluster default)

Size of a local log segment before it rolls and becomes eligible for
upload. Default **128 MiB** (`134217728`).

This is also, by default, the size of the object Redpanda writes to your
appliance - `cloud_storage_segment_size_target` (below) inherits it unless
you override it. Larger segments mean fewer, larger upload requests, which
matters most when per-request overhead is a significant fraction of total
transfer time. Against a low-latency local appliance, per-request overhead
is proportionally smaller to begin with, so this lever usually matters
less than connection concurrency does - but it's still worth testing if the
write-path benchmark shows many small, frequent uploads rather than a
smooth stream.

### `cloud_storage_segment_max_upload_interval_sec`

Idle timeout that forces a segment upload even if `segment.bytes` hasn't
been reached. Default **unset** - without it, a partition with light
traffic can hold data locally for a long time before it's eligible for
upload at all. Not relevant to a high-throughput write-path benchmark
(segments will roll on size well before this matters), but relevant to
production topics with bursty or low-volume traffic if your recovery-time
objective depends on data reaching the appliance promptly.

### `cloud_storage_segment_size_target` / `cloud_storage_segment_size_min`

Let Redpanda merge small adjacent segments in object storage into larger
ones (`cloud_storage_enable_segment_merging`, default **enabled**).
`cloud_storage_segment_size_target` defaults to your local segment size;
raising it above that triggers extra re-uploads and isn't recommended.
`cloud_storage_segment_size_min` (default 50% of local segment size)
controls the smallest segment merging will leave behind. These mostly
matter for topics that produce many small segments (low-throughput topics,
or `segment_max_upload_interval_sec` set aggressively low) - not usually a
factor in the high-throughput benchmarks in this repo, but worth knowing
about if a production topic's access pattern looks like that.

## Read-side cache and readers - tune after write/read benchmarks show where the ceiling is

### `cloud_storage_cache_size` / `cloud_storage_cache_size_percent`

Local disk cache for hydrated Tiered Storage data. Whichever of the two
(absolute bytes vs. percent of disk) is smaller wins. This cache is what
made your read-path benchmark's *second* pass through the same data fast -
which is exactly why `03-benchmark-read-path.md` has you size the backlog
to exceed it, so the benchmark measures the appliance instead of the
cache.

In production this cache is a genuine win (repeat reads of recently
hydrated data skip the appliance entirely) - just don't let it quietly
invalidate your benchmark's conclusions, and remember it eats into the
local disk headroom Tiered Storage was supposed to free up in the first
place, alongside `disk_reservation_percent` (default 20%).

### `cloud_storage_cache_chunk_size`

Size of a chunk downloaded into the cache. Default **16 MiB**. Redpanda
downloads in chunks rather than whole segments by default
(`cloud_storage_disable_chunk_reads`, default **false**) so a consumer
reading a small range doesn't pull an entire large segment. If your read
benchmark's workload reads sequentially through large ranges, whole-segment
behavior may perform differently than default chunked reads - this is a
reasonable thing to A/B if the read-path number looks lower than the raw
appliance GET latency from self-test would suggest.

### `cloud_storage_max_segment_readers_per_shard`

Maximum concurrent read cursors into hydrated segments per core (alias:
`cloud_storage_max_readers_per_shard`). Default **unset**, which resolves
to `topic_partitions_per_shard`. No restart needed. This caps how many
partitions' worth of reads a single core can service concurrently - if the
concurrency sweep in `03-benchmark-read-path.md` plateaus well before
`cloud_storage_max_connections`/hydration concurrency looks saturated,
check this property next; it's a different cap on a different resource
(read cursors, not connections).

## Properties to leave alone

`cloud_storage_upload_ctrl_*` (update interval, P/D coefficients,
min/max shares) tune the internal controller that allocates I/O and CPU
shares to the upload process dynamically. Redpanda's own docs are explicit
that you shouldn't need to touch these under normal circumstances - if
your benchmark results suggest they're the constraint, that's a case for
Redpanda support, not a config change to make unilaterally.

## Summary decision tree

1. Self-test latencies low (single-digit ms)? Start from
   `cloud_storage_max_connections` above default; latencies closer to
   public-cloud territory (tens of ms+)? Defaults are probably closer to
   already-right, and the benchmarks matter more for capacity planning.
2. Write-path benchmark plateaus with `vectorized_ntp_archiver_pending`
   climbing -> raise `cloud_storage_max_connections`; re-test.
3. Write-path benchmark shows frequent small uploads rather than a smooth
   stream -> look at `segment.bytes` and `cloud_storage_segment_max_upload_interval_sec`.
4. Read-path concurrency sweep plateaus early -> check
   `cloud_storage_max_concurrent_hydrations_per_shard` isn't pinned below
   `max_connections/2`, then `cloud_storage_max_segment_readers_per_shard`.
5. Either benchmark shows high retries rather than a clean plateau -> stop
   tuning throughput properties and revisit connectivity
   (`01-connect-and-validate.md`) and appliance-side limits first.

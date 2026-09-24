# 3. Benchmark the read path

The steps here are focused on finding the sustained throughput Redpanda can hydrate (download
and read) data back out of your appliance, isolated from the write side.

If you haven't already, do the client-placement latency check and the
throughput-based sizing pass from `02-benchmark-write-path.md` first -
both apply here too. In particular, size `partitionsPerTopic` to at least
`consumerPerSubscription` so every consumer thread gets its own partitions
instead of contending for the same ones.

## How the test isolates the read path

Reading from the batch cache or the local disk cache tells you nothing
about the appliance, so we need to ensure reads are actually forced out to
object storage.

`workloads/read-path-stress-backlog.yaml` uses OMB's `consumerBacklogSizeGB`
mechanic, so producers run alone first until the configured backlog has
accumulated. Only then will consumers start reading that backlog back.
The paired driver, `driver/driver-read-path.yaml`, sets an aggressively
small `retention.local.target.bytes` on the topic. So by the time consumers
start almost all of that backlog has already rolled off local disk and
exists only in the appliance. Consumers reading from `earliest` are then
forced to hydrate from object storage rather than serve out of local disk
or the batch cache.

`consumerBacklogSizeGB` has to comfortably exceed both the topic's total
local retention and the cluster's actual Tiered Storage cache cap,
otherwise the first pass through the data warms the local TS cache and
every read after that is a cache hit against local disk.
Rather than estimating these by hand, follow these steps to compute the proper value:

**1. Total local retention held by the topic** (`retention.local.target.bytes`
from `driver/driver-read-path.yaml` x `partitionsPerTopic` from
`workloads/read-path-stress-backlog.yaml`):

```sh
RETENTION_BYTES=$(grep 'retention.local.target.bytes=' driver/driver-read-path.yaml | cut -d= -f2)
PARTITIONS=$(awk '/^partitionsPerTopic:/{print $2}' workloads/read-path-stress-backlog.yaml)
LOCAL_RETENTION_TOTAL_BYTES=$(( RETENTION_BYTES * PARTITIONS ))
echo "Local retention total: $LOCAL_RETENTION_TOTAL_BYTES bytes"
```

**2. The cluster's actual cache cap**, run from a shell with `rpk` pointed
at your cluster. `cloud_storage_cache_size` and `cloud_storage_cache_size_percent`
are cluster properties (Redpanda uses whichever calculates smaller, in
bytes, unless one of them is `0` - see
[the property reference](https://docs.redpanda.com/current/reference/properties/object-storage-properties/#cloud_storage_cache_size_percent)).
`cloud_storage_cache_directory` is a per-broker property read from
`redpanda.yaml`, not `rpk cluster config` - SSH to a broker for that part:

```sh
CACHE_SIZE_BYTES=$(rpk cluster config get cloud_storage_cache_size)
CACHE_SIZE_PERCENT=$(rpk cluster config get cloud_storage_cache_size_percent)

# On a broker: find the cache directory (defaults to <data_directory>/cloud_storage_cache
# if cloud_storage_cache_directory is unset in redpanda.yaml), then the disk size it lives on.
CACHE_DIR=$(awk '/cloud_storage_cache_directory:/{print $2; f=1} END{if(!f) print ""}' /etc/redpanda/redpanda.yaml)
if [ -z "$CACHE_DIR" ]; then
  DATA_DIR=$(awk '/data_directory:/{print $2; exit}' /etc/redpanda/redpanda.yaml)
  CACHE_DIR="$DATA_DIR/cloud_storage_cache"
fi
DISK_BYTES=$(df --output=size -B1 "$CACHE_DIR" | tail -1 | tr -d ' ')
CACHE_CAP_FROM_PERCENT=$(( DISK_BYTES * ${CACHE_SIZE_PERCENT%.*} / 100 ))

# Redpanda's own precedence: smaller of the two, unless one is 0.
if [ "$CACHE_SIZE_BYTES" = "0" ]; then
  EFFECTIVE_CACHE_BYTES=$CACHE_CAP_FROM_PERCENT
elif [ "$CACHE_SIZE_PERCENT" = "null" ] || [ -z "$CACHE_SIZE_PERCENT" ]; then
  EFFECTIVE_CACHE_BYTES=$CACHE_SIZE_BYTES
elif [ "$CACHE_SIZE_BYTES" -lt "$CACHE_CAP_FROM_PERCENT" ]; then
  EFFECTIVE_CACHE_BYTES=$CACHE_SIZE_BYTES
else
  EFFECTIVE_CACHE_BYTES=$CACHE_CAP_FROM_PERCENT
fi
echo "Effective cache cap: $EFFECTIVE_CACHE_BYTES bytes"
```

The `awk` lines above assume a flat `key: value` line in `redpanda.yaml`,
which is the common case - if yours nests differently, just read the two
values by hand and skip straight to setting `CACHE_DIR`.

**3. Set `consumerBacklogSizeGB`** to 2x the larger of the two figures
(comfortable headroom, not a razor's-edge minimum), and write it straight
into the workload file:

```sh
FLOOR_BYTES=$LOCAL_RETENTION_TOTAL_BYTES
[ "$EFFECTIVE_CACHE_BYTES" -gt "$FLOOR_BYTES" ] && FLOOR_BYTES=$EFFECTIVE_CACHE_BYTES
TARGET_GB=$(( (FLOOR_BYTES * 2 + 999999999) / 1000000000 ))
echo "Setting consumerBacklogSizeGB: $TARGET_GB"
sed -i "s/^consumerBacklogSizeGB:.*/consumerBacklogSizeGB: $TARGET_GB/" workloads/read-path-stress-backlog.yaml
```

Fill in your connection details in `driver/driver-read-path.yaml`, then:

```sh
bin/benchmark \
  -d driver/driver-read-path.yaml \
  workloads/read-path-stress-backlog.yaml
```

Expect this run to take longer than the write-path test, since it has to produce
the entire backlog first then run OMB's warmup phase
(`warmupDurationMinutes`) before the timed consume window
(`testDurationMinutes`) even starts.

## What to watch during the run

- Consumer-side end-to-end latency and throughput from the OMB output (this is the primary read-path number)
- Redpanda's Tiered Storage cache-related metrics (hit/miss and current
  cache size). A rising hit rate mid-run means the cache is warming up and
  you're no longer purely measuring appliance reads. In that case, re-run with a larger
  backlog or a smaller cache if that happens.
- Broker CPU per core. Hydration and decompression work happens per
  shard, so an uneven per-core CPU profile can mean a small number of hot
  partitions are bottlenecking the whole test rather than the appliance.

## Reading the result, and sweeping concurrency

Run the workload a few times with `consumerPerSubscription` increased each
time (2, 4, 8, ...) while everything else stays fixed. Because each
concurrent reader can drive independent hydration requests against the
appliance, this sweep is what actually reveals your read-path concurrency
ceiling; a single-consumer run mostly measures per-request latency, not
throughput headroom.

- If throughput keeps climbing as you add consumers, `04-tuning-guide.md`'s
  read-concurrency properties (`cloud_storage_max_concurrent_hydrations_per_shard`,
  `cloud_storage_max_connections`) probably still have room before you hit
  a wall.
- If throughput flattens while consumers keep piling on, you've found a
  ceiling - check whether it's Redpanda-side (those same properties, just
  now the wrong direction: over-provisioned concurrency past what
  connections/memory can back) or appliance-side (front-end network/CPU).

Record the write-path number and this concurrency curve as iteration 0
before changing any `cloud_storage_*` property. See "The loop" at the top
of `04-tuning-guide.md`.

# 3. Benchmark the read path

Goal: find the sustained throughput at which Redpanda can hydrate (download
and read) data back out of your appliance, isolated from the write side.

## How the test isolates the read path

Reading from the batch cache or the local disk cache tells you nothing
about the appliance - you need reads that are actually forced out to
object storage.

`workloads/read-path-stress-backlog.yaml` uses OMB's `consumerBacklogSizeGB`
mechanic: producers run alone first until the configured backlog has
accumulated, and only then do consumers start, reading that backlog back.
The paired driver, `driver/driver-read-path.yaml`, sets an aggressively
small `retention.local.target.bytes` on the topic, so by the time consumers
start, almost all of that backlog has already rolled off local disk and
exists only in the appliance. Consumers reading from `earliest` are then
forced to hydrate from object storage rather than serve out of local disk
or the batch cache.

Size `consumerBacklogSizeGB` in the workload file to comfortably exceed
both the topic's local retention target and `cloud_storage_cache_size` /
`cloud_storage_cache_size_percent` (the Tiered Storage disk cache) -
otherwise the first pass through the data warms the local TS cache and
every read after that is a cache hit against local disk again, not a real
appliance round trip.

Fill in your connection details in `driver/driver-read-path.yaml`, then:

```sh
sudo bin/benchmark \
  -d driver/driver-read-path.yaml \
  workloads/read-path-stress-backlog.yaml
```

Expect this run to take longer than the write-path test: it has to produce
the entire backlog before the timed consume phase even starts.

## What to watch during the run

- Consumer-side end-to-end latency and throughput from the OMB output -
  this is your primary read-path number.
- Redpanda's Tiered Storage cache-related metrics (hit/miss and current
  cache size) - a rising hit rate mid-run means the cache is warming up and
  you're no longer purely measuring appliance reads; re-run with a larger
  backlog or a smaller cache if that happens.
- Broker CPU per core - hydration and decompression work happens per
  shard, so an uneven per-core CPU profile can mean a small number of hot
  partitions are bottlenecking the whole test rather than the appliance.

## Reading the result, and sweeping concurrency

Run the workload a few times with `consumerPerSubscription` increased each
time (2, 4, 8, ...) while everything else stays fixed. Because each
concurrent reader can drive independent hydration requests against the
appliance, this sweep is what actually reveals your read-path concurrency
ceiling - a single-consumer run mostly measures per-request latency, not
throughput headroom.

- If throughput keeps climbing as you add consumers, `04-tuning-guide.md`'s
  read-concurrency properties (`cloud_storage_max_concurrent_hydrations_per_shard`,
  `cloud_storage_max_connections`) probably still have room before you hit
  a wall.
- If throughput flattens while consumers keep piling on, you've found a
  ceiling - check whether it's Redpanda-side (those same properties, just
  now the wrong direction: over-provisioned concurrency past what
  connections/memory can back) or appliance-side (front-end network/CPU).

With write-path and read-path numbers in hand, go to `04-tuning-guide.md`
to turn them into concrete property changes.

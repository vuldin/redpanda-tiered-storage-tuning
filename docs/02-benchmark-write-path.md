# 2. Benchmark the write path

The goal of the steps outlined here is to find the sustained throughput at which Redpanda can upload segments
to your appliance, isolated from the read side.

## Before you run anything: client placement and latency

This benchmark exists to characterize your Redpanda cluster and your
appliance. Any meaningful network hop between the OMB client and the
brokers dominates the result and measures that hop instead - running the
client from a laptop, over VPN, or from a different site/region than the
brokers isn't a slower variant of this test, it's a different test that
doesn't tell you what this repo is for. This is the single most common way
to run this benchmark and get a number that looks like a Redpanda or
appliance limitation but is neither.

Check round-trip time from the machine you intend to use as the OMB client
to every broker, before touching anything else. This needs no extra
tooling - a TCP connect against the Kafka port is a solid proxy for
network RTT and works even when ICMP is blocked:

```sh
for h in broker1.example.com broker2.example.com broker3.example.com; do
  t0=$(date +%s%N)
  timeout 2 bash -c "exec 3<>/dev/tcp/$h/9092" 2>/dev/null && ok=ok || ok=FAILED
  t1=$(date +%s%N)
  printf '%-40s %6d ms (%s)\n' "$h" $(( (t1 - t0) / 1000000 )) "$ok"
done
```

Replace the hostnames and port with your own brokers and Kafka listener
port.

- **Sub-millisecond to a few ms**: same rack/AZ as the brokers - good, proceed.
- **High single digits to a few tens of ms**: there's a real network hop in
  the path (cross-AZ, a firewall/proxy, etc.) - borderline; expect some of
  this latency to show up in your results, and know that before you read
  them.
- **Tens of ms or more**: you're on a WAN link. Stop. Move the OMB client
  to a machine on the same network as the brokers before running anything
  in this repo. The appliance self-test in `01-connect-and-validate.md`
  exists specifically to characterize the appliance's own latency
  separately from this - a slow client-to-broker hop on top of that just
  measures your corporate network, not the appliance or Redpanda.

## Sizing the benchmark to your target throughput

`workloads/write-path-stress.yaml` ships with 12 partitions, 4 producer
threads, and 4 consumer threads - enough to prove the mechanics work, not
sized for any particular target throughput. Raising `producerRate` in the
workload file does not add parallelism; it only raises the ceiling a
*fixed* number of producer threads are allowed to try to hit. Actual
achievable throughput is bounded by how many producer/consumer threads you
run and how much each one can push, not by that rate field.

If you have a specific throughput target to validate against (e.g. "this
needs to sustain 500 MB/s produce"), size the workload to it instead of
guessing:

1. Run the stock benchmark once (this is your baseline from "The loop" in
   `04-tuning-guide.md` anyway). Take the achieved MB/s and divide by
   `producersPerTopic` to get a rough per-thread rate. Treat this as a
   floor, not a ceiling - client-broker latency and client CPU both limit
   per-thread throughput, and both change once you've fixed client
   placement per the check above. Re-baseline after moving the client
   before trusting this number for sizing.
2. Required thread count = `target_MB_s / per_thread_MB_s`, rounded up.
3. Set `partitionsPerTopic` to at least that many - partitions are
   Redpanda's unit of produce parallelism, so more producer threads than
   partitions just means multiple threads queuing on the same partition.
   For the read-path workload, also make sure
   `partitionsPerTopic >= consumerPerSubscription` so every consumer
   thread gets its own partitions to read instead of contending.
4. Set `producersPerTopic` (and `consumerPerSubscription` for the read
   path) to that same thread count.
5. Watch the *client's* CPU and network utilization while it runs. If the
   client itself is pegged, you're now measuring the client, not Redpanda
   or the appliance. Past a certain thread count, one machine can't drive
   enough concurrent connections - split across multiple client machines
   with OMB's distributed workers mode instead of piling more threads onto
   one box:

   ```sh
   bin/benchmark --workers http://client1:8080,http://client2:8080 \
     -d driver/driver-write-path.yaml \
     workloads/write-path-stress.yaml
   ```

   Each listed worker runs `bin/benchmark-worker` (see the OMB benchmark
   guide for the systemd/process setup). OMB has no single-remote-worker
   mode - `--workers` needs at least two entries, or omit it entirely to
   run everything embedded in one local process (fine for the initial
   mechanical pass, not for validating a real throughput target).

**Worked example** (generic numbers, not a recommendation for your case):
baseline achieves 20 MB/s with 4 producer threads -> 5 MB/s/thread. Target
is 500 MB/s. Required threads: 500 / 5 = 100. Set `partitionsPerTopic: 100`
and `producersPerTopic: 100`. If one client machine can't drive 100
threads without pegging its own CPU or NIC, split across e.g. 4 client
machines at 25 threads each via distributed workers.

## How the test isolates the upload path

`workloads/write-path-stress.yaml` runs a long, sustained produce load with
`consumerBacklogSizeGB: 0` - consumers stay caught up at the tail, reading
straight out of the batch cache, so they add negligible read-path load.
Almost everything the appliance sees is upload traffic.

The accompanying driver, `driver/driver-write-path.yaml`, enables Tiered
Storage on the topic Redpanda's OMB fork creates for the run. Fill in your
broker addresses and connection security config (see the placeholders in
the file), then run:

```sh
bin/benchmark \
  -d driver/driver-write-path.yaml \
  workloads/write-path-stress.yaml
```

Give it at least the workload's configured 35 minutes total. This is because OMB always runs a warmup phase before the timed window (`warmupDurationMinutes` + `testDurationMinutes`).
The first minute or two of the timed window will look artificially fast; this is because Redpanda still has local segments to
close and there's no upload backlog yet. So determine the steady state based off the back half of the run.

## What to watch during the run

From each broker's `/public_metrics` and `/metrics` endpoints:

- `vectorized_ntp_archiver_pending` - segments queued for upload but not
  yet uploaded. Rising and staying elevated means uploads can't keep up
  with produce; flat-and-low means the upload path has headroom.
- Redpanda's cloud storage upload metrics for error/retry rates - climbing
  retries under load point at connection exhaustion or the appliance
  throttling/erroring under concurrency, not a throughput ceiling per se.
- Broker CPU and network utilization, and (if your storage team can share
  it) the appliance's own front-end network/CPU utilization - you want to
  know which side actually saturated first.


## Reading the result

Compute achieved upload throughput as total bytes produced over the
steady-state window divided by that window's duration (or read it directly
off OMB's output/results file). Compare that number against:

- The network path's theoretical ceiling between brokers and appliance.
- What the appliance vendor rates its front-end for.

If achieved throughput is well below both of those ceilings while
`vectorized_ntp_archiver_pending` keeps climbing, the bottleneck is on
Redpanda's side. In that case go to `04-tuning-guide.md` and look at the
upload-concurrency properties first.

If achieved throughput tracks the network or appliance ceiling closely,
you've found the real limit of your environment, which is not something a Redpanda
property is going to move.

Either way, make sure to record this number before you touch any `cloud_storage_*` property since
it's iteration 0 in the tuning loop at the top of `04-tuning-guide.md`, and
every property change from here on will get compared to it.

Once you have a write-path number, move on to
`03-benchmark-read-path.md` to test the other side.

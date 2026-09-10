# 2. Benchmark the write path

Goal: find the sustained throughput at which Redpanda can upload segments
to your appliance, isolated from the read side.

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
sudo bin/benchmark \
  -d driver/driver-write-path.yaml \
  workloads/write-path-stress.yaml
```

Give it at least the workload's configured 30 minutes. The first minute or
two will look artificially fast - Redpanda still has local segments to
close and there's no upload backlog yet - so judge steady state from the
back half of the run, not the opening burst.

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
Redpanda's side of the wire - go to `04-tuning-guide.md` and look at the
upload-concurrency properties first.

If achieved throughput tracks the network or appliance ceiling closely,
you've found the real limit of your environment, not something a Redpanda
property is going to move - that's a capacity/hardware conversation, not a
tuning one.

Once you have a write-path number, move on to
`03-benchmark-read-path.md` to test the other side.

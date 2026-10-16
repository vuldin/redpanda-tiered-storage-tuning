# Isolate the write path

This is a specialized diagnostic, not the first thing to run. Start with
`03-run-realistic-workload.md` - come here if that run shows a write-
side-specific problem (upload lag climbing while produce is otherwise
healthy) and you want to isolate the upload path on its own, free of any
read-side traffic, to pin down its ceiling precisely.

If you haven't already, do the client-placement check and the
throughput-based sizing pass from `02-deploy-benchmark-clients.md` first -
both apply here too.

The goal of the steps outlined here is to find the sustained throughput at which Redpanda can upload segments
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
Redpanda's side. In that case go to `06-tuning-guide.md` and look at the
upload-concurrency properties first.

If achieved throughput tracks the network or appliance ceiling closely,
you've found the real limit of your environment, which is not something a Redpanda
property is going to move.

Either way, record this number against the baseline you already recorded
from `03-run-realistic-workload.md` - this isolation run tells you which
side is constrained, the realistic workload is still what you judge
overall progress against. See "The loop" at the top of `06-tuning-guide.md`.

Optionally, do the same for the read side via `05-isolate-read-path.md`.

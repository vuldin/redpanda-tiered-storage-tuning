# 3. Run the realistic workload

This is the benchmark to run first, and the one to keep coming back to as
you tune. `04-isolate-write-path.md` and `05-isolate-read-path.md` are
specialized tools for when this run tells you one side specifically needs
closer attention - they are not where you start.

## Why a separate workload for this

`write-path-stress.yaml` and `read-path-stress-backlog.yaml` deliberately
isolate one side of Tiered Storage (produce-only, or build-a-backlog-then-
drain-it) so you can find that side's ceiling without the other side's
traffic confusing the number. That's useful for root-causing a specific
symptom, but it's a synthetic access pattern - production traffic doesn't
pause one side while the other runs.

`workloads/realistic-workload.yaml` runs continuous produce and continuous
consume at the same time, with multiple independent consumer groups
(`subscriptionsPerTopic`) if your real traffic has more than one
application reading the topic. The goal is a result that reflects what
Tiered Storage actually does under something that looks like your
production shape, not an isolated ceiling for one side of it.

## Fill in your own numbers

Before running anything, edit `workloads/realistic-workload.yaml` and
`driver/driver-realistic-workload.yaml`:

- Size `partitionsPerTopic`, `producersPerTopic`, `subscriptionsPerTopic`,
  and `consumerPerSubscription` to your actual target throughput and
  consumer fan-out using the method in `02-deploy-benchmark-clients.md`.
- Set `producerRate` to the produce rate you actually expect in
  production - this workload's whole point is running at a realistic
  rate, not finding the client's maximum.
- Set local retention in the driver's `topicConfig` to match what you
  actually plan to run in production (or leave it unset to inherit the
  cluster default) - don't copy the deliberately-tiny value from
  `driver-read-path.yaml`, which exists only to force the read-path
  isolation test off local disk.
- Fill in your broker address and connection security config in
  `commonConfig`.

Then deploy and point enough client machines at it per
`02-deploy-benchmark-clients.md`, and run:

```sh
bin/benchmark --workers http://client1:8080,http://client2:8080 \
  -d driver/driver-realistic-workload.yaml \
  workloads/realistic-workload.yaml
```

Give it the full `warmupDurationMinutes` + `testDurationMinutes` before
reading anything - the first few minutes will look artificially good
(local segments haven't started rolling yet, no upload backlog exists),
and this workload is specifically meant to be judged at steady state, not
during the opening burst.

## Monitor the cluster <-> object storage boundary while it runs

Watching Redpanda's own produce/consume numbers tells you whether the
*client* is keeping up with your target. It doesn't tell you whether
Tiered Storage - the boundary between the cluster and your appliance - is
keeping up with Redpanda. Watch both at the same time.

`grafana/tiered-storage-detail.json` in this repo is a ready-to-import
dashboard covering exactly that boundary. Import it (Grafana: Dashboards
-> New -> Import), point its `datasource` variable at your Prometheus, and
set `Broker (instance)` to your brokers - it filters on Prometheus's own
`instance` label, so it works whether your cluster is bare-metal, VMs, or
Kubernetes. It covers:

- **Upload/download throughput** (`Cloud_storage: Sent/Receive speed to/from cloud`) -
  compare this against what you're actually producing/consuming
  (Redpanda's own produce/consume throughput metrics, or your Kafka
  topic-metrics dashboard if you have one). A large, sustained gap between
  what the client is pushing and what's moving to/from the appliance is
  the signature of a Tiered Storage bottleneck, not a client one.
- **`vectorized_ntp_archiver_pending`** (upload lag) - segments queued for
  upload but not yet uploaded. Rising and staying elevated means uploads
  can't keep up with produce; flat-and-low means the upload path has
  headroom. This number only goes down if consume/upload throughput
  exceeds produce throughput - if your workload's produce rate is set
  well above what the appliance can absorb, this will climb for as long
  as the test runs, with no self-correction.
- **`vectorized_cloud_storage_failed_uploads` / `_failed_downloads`** and
  **`vectorized_cloud_client_num_slowdowns`** - any sustained non-zero
  rate here means the appliance itself is erroring or throttling, which
  changes the diagnosis entirely (see `06-tuning-guide.md`'s connectivity
  note).
- **Chunk hydration latency / Cloud Client Lease Duration** (heatmaps) -
  per-request latency to the appliance. Compare against the single-digit-
  to-low-tens-of-ms numbers from your `rpk cluster self-test` in
  `01-connect-and-validate.md` - if request latency here is dramatically
  higher than self-test showed for the same appliance, something changed
  between an idle appliance and one under this workload's load.
- **Tiered Storage cache metrics** (object count, size) - growth here is
  expected while the workload runs; a dashboard panel plateauing while
  your client-side throughput keeps climbing is a sign the *client* is
  still the one being measured, not Tiered Storage.

Also keep an eye on the client side, not just the cluster: if a single
client machine's own CPU or network is pegged, every one of the metrics
above will look constrained even though nothing in Redpanda or the
appliance actually is. See the distributed-workers warning in
`02-deploy-benchmark-clients.md`.

## Reading the result

Record, from this run:

- Achieved produce MB/s and consume MB/s per consumer group.
- Whether `vectorized_ntp_archiver_pending` stayed flat/bounded or climbed
  for the duration of the run.
- Any non-zero `failed_uploads`/`failed_downloads`/`num_slowdowns`.

This is iteration 0 in the tuning loop at the top of `06-tuning-guide.md` -
record it before changing any `cloud_storage_*` property, since every
change from here on gets judged against it.

If this run shows a clean, bounded result on both sides, you likely don't
need the isolation tests at all - go straight to `06-tuning-guide.md` if
you still want to push further, or stop here if the result already meets
your target. If one side specifically looks constrained (upload lag
climbing with produce otherwise healthy, say), `04-isolate-write-path.md`
or `05-isolate-read-path.md` can narrow down that side on its own before
you start changing properties.

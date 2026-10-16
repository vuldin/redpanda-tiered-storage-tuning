# Tuning Redpanda Tiered Storage for Self-Hosted Object Storage

A runbook and benchmark kit for operators running Redpanda Tiered Storage
against self-hosted, S3-compatible object storage to find the optimal tuning for Redpanda when using an on-prem appliance
(VAST Data, Dell ECS/PowerScale, NetApp StorageGRID, Pure FlashBlade,
Cloudian, Ceph RGW, MinIO, and similar) rather than a public cloud provider
like AWS S3 or GCS.

## Why this is a different tuning problem than "S3"

Most Tiered Storage guidance (including Redpanda's own docs) is written
against public cloud object storage, where round-trip latency to the object
store is commonly tens of milliseconds and the provider enforces its own
rate limits. Redpanda's default `cloud_storage_*` settings are calibrated
for that profile.

But an on-prem appliance can have round-trip
latency of single-digit milliseconds, there's no external rate limiter,
and the constraints are whatever your network and the appliance's own
front-end can sustain. Defaults tuned to avoid overwhelming a rate-limited public endpoint can leave a fast local appliance under-utilized.

Request concurrency (`cloud_storage_max_connections` and other similar variables) will likely provide the most tuning benefit.
But also keep in mind that the appliance's own characteristics (multipart-upload cost, TLS/cert setup) matter more than they do against a cloud provider that everyone has already tuned against.

This repo does not publish a single set of "correct" tuning values since your
appliance, network, and workload will determine those. It also isn't only
about finding a raw ceiling at the cluster-to-object-storage boundary -
the primary benchmark is meant to look like your actual production
traffic, not an isolated best case. Instead it gives you:

1. A validation step to confirm Redpanda can reach the appliance correctly.
2. A way to deploy as many benchmark client machines as your target
   throughput actually needs, instead of one box quietly becoming the
   real bottleneck.
3. A realistic, mixed produce+consume workload you size to your own
   expected traffic, plus two specialized workloads that isolate the
   write or read path on its own when you need to narrow down which side
   a problem is on.
4. A ready-to-import dashboard and metric list for watching the
   cluster-to-object-storage boundary while a workload runs.
5. A tuning guide that maps what you see in a run to the specific
   property you'd change in response.

## Prerequisites

- A Redpanda cluster (26.1+ recommended) with network access to your object
  storage appliance's S3-compatible endpoint.
- One or more machines, on the same network as the brokers, to run as OMB
  benchmark clients - see `docs/02-deploy-benchmark-clients.md` for how
  many you'll likely need.
- [Redpanda's OMB fork](https://github.com/redpanda-data/openmessaging-benchmark)
  built per Redpanda's
  [benchmark guide](https://docs.redpanda.com/current/develop/benchmark/)
  on every client machine.
- `rpk` configured against the cluster.
- A Grafana + Prometheus stack scraping the cluster, if you want to use
  the dashboard in `grafana/`. Not required to run the benchmarks
  themselves.

## Quickstart

1. **Validate the appliance.** Point Redpanda's `cloud_storage_*`
   properties at your appliance and run `rpk cluster self-test
   --only-cloud-test` to confirm it's reachable with healthy latency,
   before you draw any conclusions from a benchmark.
   -> `docs/01-connect-and-validate.md`
2. **Deploy the benchmark client(s).** Check client-to-broker latency
   from every machine you intend to use as a client, size the thread
   count you need against your target throughput, and - if one machine
   can't drive that many threads - deploy OMB's distributed workers
   across multiple client machines.
   -> `docs/02-deploy-benchmark-clients.md`
3. **Run the realistic workload.** Fill in `workloads/realistic-workload.yaml`
   and `driver/driver-realistic-workload.yaml` with your own target
   produce rate, consumer fan-out, and connection details, run it, and
   watch the cluster-to-object-storage boundary with `grafana/tiered-storage-detail.json`
   while it runs. This is the benchmark to run first and the one to keep
   coming back to.
   -> `docs/03-run-realistic-workload.md`
4. **Tune.** Use the realistic workload's result as your baseline and
   work through the properties one at a time, re-running the same
   workload after each change.
   -> `docs/06-tuning-guide.md`
5. **(Optional) Isolate a side.** If step 3 pointed at a write- or
   read-specific problem, isolate that path on its own, free of the
   other side's traffic, before you keep tuning.
   -> `docs/04-isolate-write-path.md` / `docs/05-isolate-read-path.md`

## Repo layout

```
docs/
  01-connect-and-validate.md    Point Redpanda at the appliance; self-test
  02-deploy-benchmark-clients.md  Client placement, sizing, distributed workers
  03-run-realistic-workload.md  The primary benchmark: realistic traffic + monitoring
  04-isolate-write-path.md      Specialized: find your write ceiling alone
  05-isolate-read-path.md       Specialized: find your read ceiling alone
  06-tuning-guide.md            Property-by-property tuning decision tree
driver/
  driver-realistic-workload.yaml  OMB driver-redpanda config for the primary workload
  driver-write-path.yaml         OMB driver-redpanda config for the write isolation test
  driver-read-path.yaml          OMB driver-redpanda config for the read isolation test
workloads/
  realistic-workload.yaml        OMB workload: mixed produce+consume, size to your traffic
  write-path-stress.yaml         OMB workload: sustained produce, TS on, no backlog
  read-path-stress-backlog.yaml  OMB workload: builds a backlog, then reads it back
scripts/
  benchmark-worker.service      systemd unit for an OMB distributed worker
grafana/
  tiered-storage-detail.json    Dashboard for the cluster<->object-storage boundary
```

Start with `docs/01-connect-and-validate.md`.

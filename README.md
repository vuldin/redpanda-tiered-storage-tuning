<title>Tuning Redpanda Tiered Storage for Self-Hosted Object Storage</title>

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
appliance, network, and workload will determine those. Instead it gives you the following tools:

1. A validation step to confirm Redpanda can reach the appliance correctly.
2. Two benchmark workloads (write path / upload, and read path / hydration)
   that isolate each side of Tiered Storage so you can measure your own
   ceiling instead of guessing.
3. A tuning guide that maps what you'll see in each benchmark to the
   specific property you'd change in response.

## Repo layout

```
docs/
  01-connect-and-validate.md   Point Redpanda at the appliance; self-test
  02-benchmark-write-path.md   Stress the upload path, find your write ceiling
  03-benchmark-read-path.md    Stress the hydration path, find your read ceiling
  04-tuning-guide.md           Property-by-property tuning decision tree
driver/
  driver-write-path.yaml       OMB driver-redpanda config for the write test
  driver-read-path.yaml        OMB driver-redpanda config for the read test
workloads/
  write-path-stress.yaml       OMB workload: sustained produce, TS on, no backlog
  read-path-stress-backlog.yaml OMB workload: builds a backlog, then reads it back
```

## Prerequisites

- A Redpanda cluster (26.1+ recommended) with network access to your object
  storage appliance's S3-compatible endpoint.
- [Redpanda's OMB fork](https://github.com/redpanda-data/openmessaging-benchmark)
  built per Redpanda's
  [benchmark guide](https://docs.redpanda.com/current/develop/benchmark/),
  with a benchmark driver host that can reach the cluster.
- `rpk` configured against the cluster.

Start with `docs/01-connect-and-validate.md`.

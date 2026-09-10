<title>Tuning Redpanda Tiered Storage for Self-Hosted Object Storage</title>

# Tuning Redpanda Tiered Storage for Self-Hosted Object Storage

A runbook and benchmark kit for operators running Redpanda Tiered Storage
against **self-hosted, S3-compatible object storage** - an on-prem appliance
(VAST Data, Dell ECS/PowerScale, NetApp StorageGRID, Pure FlashBlade,
Cloudian, Ceph RGW, MinIO, and similar) rather than a public cloud provider
like AWS S3 or GCS.

## Why this is a different tuning problem than "S3"

Most Tiered Storage guidance (including Redpanda's own docs) is written
against public cloud object storage, where round-trip latency to the object
store is commonly tens of milliseconds and the provider enforces its own
rate limits. Redpanda's default `cloud_storage_*` settings are calibrated
for that profile.

A local, on-prem appliance usually looks nothing like that: round-trip
latency can be single-digit milliseconds, there's no external rate limiter,
and the constraint is whatever your network fabric and the appliance's own
front-end can sustain. That changes the tuning direction:

- Defaults tuned to avoid overwhelming a rate-limited public endpoint can
  leave a fast local appliance under-utilized.
- The lever that usually matters most is **request concurrency**
  (`cloud_storage_max_connections` and friends), not object size - on a
  low-latency link you don't need huge objects to amortize round-trip cost.
- The appliance's own characteristics (multipart-upload cost, path-style vs
  virtual-hosted addressing, TLS/cert setup) matter more than they do
  against a hyperscaler that everyone has already tuned against.

This repo does not publish a single set of "correct" tuning values - your
appliance, network, and workload determine those. Instead it gives you:

1. A validation step to confirm Redpanda can reach the appliance correctly
   before you draw any performance conclusions.
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

# 1. Connect and validate

## Point Redpanda at the appliance

The properties below are the ones that most commonly need a non-default
value when the object store is self-hosted rather than a public cloud
provider. Set them with `rpk cluster config edit` (or `rpk cluster config
set <property> <value>` one at a time). Most require a broker restart -
check the "Requires restart" column in each linked reference page before
you plan the change.

| Property | Purpose | Default |
|---|---|---|
| [`cloud_storage_enabled`](https://docs.redpanda.com/current/reference/properties/object-storage-properties/#cloud_storage_enabled) | Turns Tiered Storage on cluster-wide. | `false` |
| [`cloud_storage_api_endpoint`](https://docs.redpanda.com/current/reference/properties/object-storage-properties/#cloud_storage_api_endpoint) | The appliance's S3-compatible endpoint hostname. Public-cloud defaults (auto-derived from region/bucket) don't apply here - set this explicitly. | `null` |
| [`cloud_storage_api_endpoint_port`](https://docs.redpanda.com/current/reference/properties/object-storage-properties/#cloud_storage_api_endpoint_port) | Override if the appliance doesn't serve S3 on 443. | `443` |
| [`cloud_storage_url_style`](https://docs.redpanda.com/current/reference/properties/object-storage-properties/#cloud_storage_url_style) | `path` or `virtual_host` addressing. Left unset, Redpanda tries to auto-detect (path-style for MinIO, virtual-host-first for AWS), but many on-prem appliances only support one style. Check your appliance's S3 documentation and set this explicitly rather than relying on auto-detection. | `null` |
| [`cloud_storage_disable_tls`](https://docs.redpanda.com/current/reference/properties/object-storage-properties/#cloud_storage_disable_tls) | Set `true` only if TLS is terminated in front of the appliance (e.g. a load balancer) and the connection from Redpanda itself is plaintext. | `false` |
| [`cloud_storage_trust_file`](https://docs.redpanda.com/current/reference/properties/object-storage-properties/#cloud_storage_trust_file) | Path to a CA cert if the appliance presents a self-signed or internal-CA certificate, which is common for on-prem gear. Leave unset to use the OS CA pool. | `null` |
| [`cloud_storage_credentials_source`](https://docs.redpanda.com/current/reference/properties/object-storage-properties/#cloud_storage_credentials_source) / [`cloud_storage_access_key`](https://docs.redpanda.com/current/reference/properties/object-storage-properties/#cloud_storage_access_key) / `cloud_storage_secret_key` | Static credentials are the norm for on-prem appliances - there's usually no equivalent to IAM instance-profile auth. | `null` |

If the appliance's S3 gateway sits behind a load balancer or proxy, confirm
with your storage team whether it's doing TLS termination and whether it
enforces its own connection limits - those can quietly become your real
ceiling before any Redpanda property does.

## Enable Tiered Storage on a topic

Two mechanisms exist; use the modern one unless you're on a pre-26.1
cluster:

```sh
# 26.1+: per-topic storage mode
rpk topic create my-topic -c redpanda.storage.mode=tiered

# or cluster-wide, so every new topic inherits it with zero per-topic config
rpk cluster config set default_redpanda_storage_mode tiered

# pre-26.1: legacy remote read/write pair
rpk topic create my-topic -c redpanda.remote.write=true -c redpanda.remote.read=true
```

Enable Tiered Storage **before** producing to a topic. Enabling it on a
topic that already has data starts uploading from the earliest local
offset, and toggling it off and back on again is explicitly not
recommended - both docs and Redpanda support flag this as a source of data
gaps.

## Validate the appliance with `rpk cluster self-test`

`rpk cluster self-test` has three independent test families: disk,
network, and cloud storage. Only the cloud storage test exercises the
object storage endpoint - the disk test benchmarks local NVMe/SSD and says
nothing about the appliance.

```sh
rpk cluster self-test start --only-cloud-test
```

This runs a real round trip against your configured bucket on every
participating broker: put a small object, list, get, head, delete, then a
bulk put+delete and a multipart put.

**Ordering matters.** The cloud storage test only runs when
`cloud_storage_enabled=true`, so you cannot validate the appliance before
configuring Tiered Storage - you have to enable Tiered Storage first, then
self-test. Trying to self-test the appliance as a pre-flight check before
touching any `cloud_storage_*` property will not run the cloud test at all.

### Reading the results

Watch the per-operation `AVG DURATION` on every node, not just node 0 -
appliance-side load balancing or an unevenly-loaded network path can make
one broker's view of the appliance look very different from another's.

A few things worth specifically checking:

- **Multipart put is usually the slowest operation by a wide margin**, even
  against a healthy appliance - it's doing more work (initiate, upload
  parts, complete) than a single PUT. Don't compare it directly to the
  single-object put/get/head numbers.
- If put/get/head latencies are in the low tens of milliseconds or higher,
  the properties tuned for public-cloud latency (see the tuning guide) are
  probably already close to right for you, and the read/write benchmarks
  below will matter more for capacity planning than for tuning direction.
- If they're single-digit milliseconds, that's a strong signal you're
  latency-rich relative to what Redpanda's defaults assume, and the
  concurrency-focused tuning direction in `04-tuning-guide.md` is likely to
  apply.

Once the appliance validates cleanly, move on to
`02-benchmark-write-path.md`.

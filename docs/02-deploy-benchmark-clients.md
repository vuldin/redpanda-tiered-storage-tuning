# 2. Deploy the benchmark clients

Every workload in this repo is only as good as the machine(s) running it.
This step covers where those machines need to live, how many you need,
and how to actually start more than one.

## Client placement and latency

This benchmark exists to characterize your Redpanda cluster and your
appliance. Any meaningful network hop between the OMB client and the
brokers dominates the result and measures that hop instead - running the
client from a laptop, over VPN, or from a different site/region than the
brokers isn't a slower variant of this test, it's a different test that
doesn't tell you what this repo is for. This is the single most common way
to run any workload in this repo and get a number that looks like a
Redpanda or appliance limitation but is neither.

Check round-trip time from every machine you intend to use as an OMB
client to every broker, before touching anything else. This needs no
extra tooling - a TCP connect against the Kafka port is a solid proxy for
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
- **Tens of ms or more**: you're on a WAN link. Stop. Move every OMB
  client to a machine on the same network as the brokers before running
  anything in this repo. The appliance self-test in
  `01-connect-and-validate.md` exists specifically to characterize the
  appliance's own latency separately from this - a slow client-to-broker
  hop on top of that just measures your corporate network, not the
  appliance or Redpanda.

## Sizing: how many threads, and how many machines

Every workload file in this repo ships with a handful of producer/consumer
threads and partitions - enough to prove the mechanics work, not sized
for any particular throughput target. Raising a workload's `producerRate`
does not add parallelism; it only raises the ceiling a *fixed* number of
threads are allowed to try to hit. Actual achievable throughput is bounded
by how many producer/consumer threads you run and how much each one can
push, not by that rate field.

If you have a specific throughput target to validate against (e.g. "this
needs to sustain 500 MB/s produce, read by two downstream consumer
groups"), size the workload to it instead of guessing:

1. Run a workload once as shipped (this doubles as iteration 0 in the
   tuning loop in `06-tuning-guide.md`). Take the achieved MB/s and divide
   by the thread count (`producersPerTopic` for produce,
   `consumerPerSubscription` for consume) to get a rough per-thread rate.
   Treat this as a floor, not a ceiling - client-broker latency and
   client CPU both limit per-thread throughput, and both change once
   you've fixed client placement per the check above. Re-baseline after
   moving the client before trusting this number for sizing.
2. Required thread count = `target_MB_s / per_thread_MB_s`, rounded up -
   worked out separately for produce and for each consumer group you
   care about.
3. Set `partitionsPerTopic` to at least the producer thread count -
   partitions are Redpanda's unit of produce parallelism, so more
   producer threads than partitions just means multiple threads queuing
   on the same partition. Also keep `partitionsPerTopic >= consumerPerSubscription`
   so every consumer thread gets its own partitions to read instead of
   contending for the same ones.
4. Set `producersPerTopic` and `consumerPerSubscription` to the thread
   counts from step 2.
5. Watch *client* CPU and network utilization while it runs, on every
   client machine. If a client itself is pegged, you're now measuring
   the client, not Redpanda or the appliance - move to distributed mode
   below rather than add more threads to an already-saturated box.

**Worked example** (generic numbers, not a recommendation for your case):
baseline achieves 20 MB/s with 4 producer threads -> 5 MB/s/thread. Target
is 500 MB/s. Required threads: 500 / 5 = 100. Set `partitionsPerTopic: 100`
and `producersPerTopic: 100`.

## Running more than one client: distributed workers

A single machine can only drive so many concurrent connections before its
own CPU or NIC becomes the limit - at that point, every thread you add
makes the number worse, not better, because you're measuring the client.
Past that point, split the thread count the sizing pass gave you across
multiple client machines instead of piling more threads onto one box.

**This matters even at thread counts that sound modest.** A single
embedded `bin/benchmark` process (no `--workers`) runs *every* producer
and consumer thread from steps above in one JVM, on one machine - the
same box doing all of the produce work and all of the consume work. If
that box's CPU or network is the actual constraint, every connection it
opens, to every broker, shows the same symptom: throughput plateaus well
below target while the cluster itself looks idle. Don't assume a single
client machine is fine just because the thread count looks small on paper
- check that machine's own CPU/network while a run is active.

### Set up each client machine

On every machine you'll use as a client:

1. Install a JDK and build [Redpanda's OMB fork](https://github.com/redpanda-data/openmessaging-benchmark)
   per Redpanda's [benchmark guide](https://docs.redpanda.com/current/develop/benchmark/),
   or copy over an already-built `/opt/benchmark` directory from one
   machine to the others - the build output isn't machine-specific.
2. Install `scripts/benchmark-worker.service` from this repo as a systemd
   unit (replace `User=REPLACE_ME` with whatever user owns `/opt/benchmark`
   on that host):

   ```sh
   sudo cp scripts/benchmark-worker.service /etc/systemd/system/
   sudo sed -i "s/REPLACE_ME/$(whoami)/" /etc/systemd/system/benchmark-worker.service
   sudo systemctl daemon-reload
   sudo systemctl enable --now benchmark-worker
   ```

3. Confirm the worker is listening before moving on:

   ```sh
   curl -sf http://localhost:8080/ >/dev/null && echo "worker up" || echo "worker NOT responding"
   ```

Repeat on every client machine. They don't need to be identical, but they
do all need to pass the latency check above against your brokers.

### Run a workload against all of them

From any one machine that can reach every worker's port 8080 (this can be
one of the worker machines itself, or a separate coordinator host - it
only orchestrates, it doesn't need to be beefy):

```sh
bin/benchmark --workers http://client1:8080,http://client2:8080,http://client3:8080 \
  -d driver/driver-realistic-workload.yaml \
  workloads/realistic-workload.yaml
```

- `--workers` needs at least two entries - OMB has no single-remote-worker
  mode. List every client machine's worker address, comma-separated, no
  spaces.
- Omitting `--workers` entirely runs everything embedded in one local
  process instead. That's fine for confirming the mechanics work once,
  but see the warning above before trusting any throughput number from
  that mode.
- OMB spreads the configured producer/consumer threads across however
  many workers you listed - it doesn't matter which worker runs which
  thread, only that the total thread count matches what your sizing pass
  in the previous section called for.

Once your clients are deployed and sized, move on to
`03-run-realistic-workload.md`.

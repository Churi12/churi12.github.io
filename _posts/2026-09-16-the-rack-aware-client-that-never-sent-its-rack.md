---
layout: post
title: "The rack-aware client that never sent its rack"
date: 2026-09-16
author: Miguel Santos
tags: [mimir]
---

Someone ran Mimir on ingest storage with a rack-aware Kafka, three replicas per partition spread across three availability zones, and `-ingest-storage.kafka.client-rack` set on every ingester. They then measured their cross-AZ traffic and found it sitting at a flat two thirds. Which is exactly the number you get if the rack setting does nothing at all, because with RF=3 and randomly placed leaders, two out of every three reads leave the zone.

The flag was not broken. It was being passed to the Kafka client correctly, all the way down. It just never reached the wire.

## Where the rack goes

`commonKafkaClientOptions` turns the config into a franz-go option:

```go
kgo.Rack(cfg.ClientRack)
```

That is the right call and it works. `kgo.Rack` makes franz-go put the rack in the fetch requests that *franz-go itself* builds, and honour the `PreferredReadReplica` the broker sends back. That is KIP-392, rack-aware fetching from the closest replica instead of the leader, and it is entirely a property of the client's own consumer loop.

Mimir's partition reader does not use franz-go's consumer loop. With `-ingest-storage.kafka.fetch-concurrency-max` above zero, which is the default of 12, it pauses client-side fetching and hands the partition to its own `ConcurrentFetchers`, which resolves the leader and assembles fetch requests by hand:

```go
req := kmsg.NewFetchRequest()
```

Nothing on that request is ever set to the rack. And on the way back, the preferred replica the broker offers is thrown away, with a comment in `parseFetchResponse` saying so in as many words. So on the default read path the rack is never sent, every fetch goes to the partition leader, and rack-aware consumption never engages at all.

Two independent code paths, both correct in isolation, and a config flag that quietly means nothing once you use the fast one.

## Why it is worse than a silent no-op

A silent no-op you have to opt into is a documentation bug. This one you opt into by default. The Helm chart and the jsonnet both turn the flag on for zone-aware ingesters: `ingester.zoneAwareReplication.autoIngestStorageClientRack` and `ingest_storage_set_client_rack` each default to true. And the chart's own values comment tells you what you are getting:

```yaml
# -- When true and zone-aware replication is enabled, automatically set the
# -ingest-storage.kafka.client-rack flag on each ingester zone to its zone name.
# This enables Kafka rack-aware consumption so that ingesters prefer reading
# from Kafka replicas in the same availability zone, reducing cross-AZ traffic.
autoIngestStorageClientRack: true
```

Every sentence after the first is untrue with the default fetch concurrency. So the operator does nothing, gets the flag set for them, reads a comment promising a cost saving, and receives no change whatsoever. There is no error, no warning, no metric that moves. The only way to find out is to go and measure your inter-zone bytes and then read enough of the fetch path to work out why the number did not budge. That is what the reporter did, and it is a lot to ask.

## Not fixing it

The obvious fix is to make it true: set `Rack` on the concurrent fetch request and honour `PreferredReadReplica`. I decided not to, and being explicit about why was most of the PR.

Setting `Rack` on its own moves exactly zero traffic. Under KIP-392 the broker does not serve the fetch from the closest replica for you. It answers with a preferred replica id and the consumer has to re-issue its fetches against that broker. Without the redirect, adding the field is a pure no-op that looks like a fix, which is worse than the current state because now the code reads as if it handles racks.

Doing the redirect properly means tracking a preferred replica per in-flight fetch alongside the leader epoch, coping with a follower whose high watermark lags the leader's, and resetting back to the leader on `OFFSET_NOT_AVAILABLE`, `NOT_LEADER_OR_FOLLOWER` and fenced epochs, plus the replica lease expiry that franz-go handles inside its own consumer and which `ConcurrentFetchers` would now have to reimplement. That is a real change to the hot path that ingests every metric in the cluster. Folding a speculative version of it into a bug fix is how you turn a documentation problem into an outage.

So the PR changes no behaviour. It makes the limitation impossible to hit silently: a warning when a rack is configured with concurrent fetching, and the same statement on the flag help, the generated config reference, the chart values comment and the jsonnet comment.

## Getting the warning in the wrong place first

I put the warning in `NewKafkaReaderClient`, which is where the rack becomes a client option. That reads like the natural home for it and it is wrong twice over.

That constructor has nine callers, and most of them never build `ConcurrentFetchers`. The probe client in `fetchRecordTimestampAtOffset` calls `PollRecords`, which is franz-go's own fetch path, so it *does* honour the rack. Warning there does not merely add noise, it states something false. The offset reader client fetches no records at all. On top of which, an ingester builds several of these during startup, so the warning fired about three times before the process was even ready.

It belongs in the one branch where the rack is genuinely dead, in `SingleClusterPartitionReader.start`:

```go
if r.kafkaCfg.FetchConcurrencyMax > 0 {
    if r.kafkaCfg.ClientRack != "" {
        level.Warn(r.logger).Log(
            "msg", "the configured Kafka client rack has no effect because concurrent fetching is enabled: records are always fetched from the partition leader, regardless of the configured rack",
            ...
```

Same condition the code already branches on to pause client fetching, so the warning cannot drift out of sync with the behaviour it describes. Once per partition reader.

The wording needed the same treatment. My first version said the flag does nothing on the read path, full stop. That is false for the usage tracker, which calls `PollFetches` and `PollRecords` straight on the client and has no `ConcurrentFetchers` anywhere in the package. The Kafka config is shared across components, so a sentence in the flag help is read by operators of every one of them, and a confident overgeneralisation there misinforms people whose setup works fine. It now says the rack is ignored when consuming a partition with concurrent fetching enabled, which is the true statement.

## The sentence I deleted

The warning originally ended with advice: set fetch concurrency to zero to read from the closest replica. It is accurate, and I took it out.

Fetch concurrency of 12 is the default because it is faster. Telling an operator to drop it to zero is telling them to trade ingest throughput for cross-AZ cost, and the right answer depends on their data volume, their instance types and what their cloud provider charges for inter-zone traffic. A startup log line knows none of that. It can tell you what is happening and leave the trade to the person who can price it.

## The lesson

There were two `ConcurrentFetchers` construction sites, not one. I only found the second, in `pkg/blockbuilder`, because I went looking for every non-test caller instead of trusting the conclusion I had already written down. The rack is ignored there too. I left it without its own log line, since block-builder walks many partitions per cycle and it would be noisy, but I said so in the PR rather than letting a reviewer discover the gap.

And moving code invalidates prose you have already published. Relocating that warning falsified four separate claims in a PR description that was already open: where the warning lived, which file the test was in, what the flag help said, and a promise about a changelog placeholder that had already been dealt with. Nobody rejects a PR for a stale description, which is exactly why it is worth going back and fixing.

The change is in [grafana/mimir#16605](https://github.com/grafana/mimir/pull/16605).

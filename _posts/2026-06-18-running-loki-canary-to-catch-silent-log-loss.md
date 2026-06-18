---
layout: post
title: "Running loki-canary to catch the logs Loki silently drops"
date: 2026-06-18
author: Miguel Santos
tags: [loki]
---

The failure mode that scares me most with a logging stack is not the loud one. A crashed ingester pages you. The quiet one — Loki accepting writes, returning 200s, and silently dropping a fraction of lines somewhere between push and store — does not, and you only find out when you go looking for a log during an incident and it is not there. `loki-canary` exists to make that quiet failure loud, and it is underused mostly because its docs were thin. I spent some time fixing those docs upstream, so here is the practical version.

## What the canary actually does

`loki-canary` is a tiny standalone binary that runs next to Loki and does one thing: it continuously writes log lines with known timestamps, then reads them back and checks they all arrived, in order, on time. It exposes the results as Prometheus metrics. The clever part is that it reads back two ways — over a live WebSocket tail *and* with a direct query — so it can distinguish "the log never arrived" from "the log arrived but the tail missed it."

The metrics that matter:

- `loki_canary_missing_entries_total` — lines that were written but never read back. This climbing is the alarm. It means real log loss.
- `loki_canary_unexpected_entries_total` — lines read back that were not expected (often out-of-order or duplicates).
- `loki_canary_response_latency` — a histogram of how long the round trip took.

You alert on `missing_entries` climbing, and you now know about silent loss in seconds instead of during an incident.

## Running it

It ships as a binary on the [Loki releases](https://github.com/grafana/loki/releases) page and as a container image, `grafana/loki-canary`, tagged to match your Loki version. The minimal run just needs an address and a unique label per instance:

```bash
loki-canary \
  -addr=loki:3100 \
  -labelname=instance \
  -labelvalue=loki-canary-1
```

By default the canary only *reads* over the WebSocket — an agent like Grafana Alloy is still responsible for shipping the canary's stdout into Loki. If you want the canary to push its own logs directly, which is simpler for a quick check, set `-push`:

```bash
loki-canary \
  -addr=loki.example.com:3100 \
  -push=true \
  -user=canary \
  -pass="$LOKI_PASSWORD" \
  -tenant-id=team-a \
  -labelname=instance \
  -labelvalue=loki-canary-team-a
```

A small detail that trips people up: when `-push=true`, the default `-streamvalue` flips from `stdout` to `push`. If your read query is hardcoded to one, that matters.

## The misconfigurations that waste an afternoon

Almost every "the canary reports everything as missing" report comes down to one of these:

- **Label mismatch.** The canary filters its read query with `-labelname`/`-labelvalue` and `-streamname`/`-streamvalue`. Whatever agent ships its logs into Loki *must* apply the exact same labels, or the read query matches nothing and every line looks lost. Confirm by running the canary's own query (for example `{name="loki-canary", stream="stdout"}`) directly in Grafana or `logcli`.
- **Shared label value across instances.** Each canary needs a unique `-labelvalue`. Two canaries with the same value read each other's lines and report phantom out-of-order and unexpected entries.
- **`-labels` overrides the simple flags.** If you set the comma-separated `-labels`, it overrides `-labelname` and `-streamname` entirely. Easy to set both and wonder why the selector is wrong.
- **TLS without the port.** Enabling `-tls=true` switches the connection to `wss://`, so `-addr` must point at the TLS port. And supplying `-cert-file`/`-key-file`/`-ca-file` without `-tls=true` fails fast with `Must set --tls when specifying client certs`.

## Tuning so you do not DoS yourself

Be careful with the relationship between `-interval` (how often it writes) and `-pruneinterval` (how often it queries Loki for anything missing over the WebSocket). A high write rate plus a short prune interval means the canary hammers Loki with direct queries, and those are capped at 1000 results — so with many canaries you can both miss logs *and* generate a small denial-of-service against your own cluster. The defaults are sane; change them deliberately.

If `unexpected_entries` is high rather than `missing`, the usual fix is raising `-wait` (default `1m`) so the canary gives WebSocket entries longer to arrive before it goes looking for them.

## Why bother

A logging system you cannot trust is worse than no logging system, because you make decisions on it during incidents. The canary is maybe ten minutes to deploy and turns "I hope Loki is not dropping anything" into a metric you can alert on. The full flag reference and troubleshooting guide now live in the [loki-canary docs](https://grafana.com/docs/loki/latest/operations/loki-canary/) — I sent the expansion upstream after hitting the gaps myself.

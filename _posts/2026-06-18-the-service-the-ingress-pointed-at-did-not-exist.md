---
layout: post
title: "When zone-aware Mimir pointed its ingress at a service that was never created"
date: 2026-06-18
author: Miguel Santos
tags: [mimir]
---

Someone hit a sharp edge in the Mimir Helm chart: turn on zone-aware replication for the alertmanager and the ingress starts returning errors, because it routes to a service the chart never creates. I traced it through the templates and fixed it the way a maintainer had already said it should be fixed. It is a good small case study in how a sensible default for one component quietly breaks another.

## The symptom

With `alertmanager.zoneAwareReplication.enabled=true`, the ingress for the alertmanager paths breaks:

```
error: services "mimir-alertmanager" not found
```

The paths are there, the ingress is there, but the backend service name it points at does not resolve. Annoying, and not obvious why, because nothing in your values mentions that service name.

## Why the service was missing

Zone-aware replication runs one set of pods per zone, so instead of a single deployment the chart renders per-zone workloads and per-zone services: `mimir-alertmanager-zone-a`, `-zone-b`, `-zone-c`, plus a headless service for gossip. There is helper logic that decides which services get created, and it only adds the plain, zone-less `mimir-alertmanager` service in specific cases — notably while a migration is in progress.

So in the steady state — zone-aware on, no migration — you get the per-zone services and the headless one, but not the plain `mimir-alertmanager`. Meanwhile the ingress template hardcodes the stable name `mimir-alertmanager` for its backend. The two assumptions disagree. One side stopped creating the name the other side still depends on.

This was a relatively recent regression: a change had repointed the default ingress routes from the headless service to a regular ClusterIP one, which is the right call in general, but it did not account for the zone-aware layout where that ClusterIP service is not always rendered.

## Two ways to fix it, and the one that is right

The tempting fix is to make the ingress fall back to the headless service when zone-aware is on. It is a two-line change. Someone had already proposed exactly that in an earlier pull request.

A maintainer pushed back, and they were right: an ingress should route to a regular ClusterIP service, not a headless one. Headless services exist to expose individual pod IPs for things like gossip, not to be a stable load-balanced endpoint behind an ingress. So the correct fix is not to bend the ingress toward the headless service — it is to make the missing ClusterIP service exist.

That is what I did. When zone-aware replication is enabled (and migration is not running), render a plain `mimir-alertmanager` ClusterIP service whose selector matches the alertmanager pods across all zones. The stable name resolves again, and it load-balances over every zone the way an ingress backend should.

Two details mattered:

- **It must select across all zones.** The per-zone services pin a `zone` label in their selector. The new service deliberately omits it, so it covers every zone's pods under one name.
- **It must not double-count metrics.** A ServiceMonitor scrapes services by label, and the per-zone services already expose those pods. If the new service were also scraped you would get duplicate series, so it carries the same opt-out label the headless service uses. The per-zone services keep owning metrics; this one only does routing.

I guarded it so it only renders in the case that was actually broken. With zone-awareness off, or mid-migration, the existing template already creates the plain service, so adding another would be a duplicate. I rendered all three cases with `helm template` and checked the chart's golden-record manifests to confirm the new service shows up only where it should and nothing else moved.

## If you run Mimir with zone-aware alertmanager

If you turned on zone-aware replication for the alertmanager and your ingress routes broke with a missing-service error, this is why, and the fix is to have the stable ClusterIP service exist rather than to point the ingress somewhere it should not go. More generally: when a chart splits one component into per-zone copies, check that every stable name other resources reference still gets rendered. A default that is correct for the single-zone layout can silently drop something the multi-zone layout still needs.

The change is in [grafana/mimir#15740](https://github.com/grafana/mimir/pull/15740).

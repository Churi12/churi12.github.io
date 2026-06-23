---
layout: post
title: "ExternalDNS created the AAAA record, then forgot it existed"
date: 2026-06-23
author: Miguel Santos
tags: [external-dns]
---

ExternalDNS is a reconcile loop. Every minute it reads the records that exist at your DNS provider, reads the records your Kubernetes resources say should exist, diffs the two, and applies the difference. That loop only converges if the read side and the write side agree on what a record is. If the controller can create something it cannot later see, it will create it forever. That is exactly what was happening to AAAA records on the DNSimple provider, and the fix is one line.

## The symptom

Someone reported that ExternalDNS kept trying to create an AAAA record that already existed. It had created the record correctly on an earlier run, and now every reconcile it tried to create the same record again and the provider rejected it as a duplicate. A record that should have settled into a quiet no-op was instead a permanent error in the logs.

## Why a working create turns into a loop

The giveaway is that the create worked. The record was there. So the bug could not be in the create path, it had to be in how the controller perceived the current state on the next pass.

I went to read the provider. ExternalDNS providers implement two halves that have to mirror each other: `Records()` reads the current state back from the provider, and the apply path turns the planned diff into API calls. In the DNSimple provider the apply path does not filter by record type at all, it sends whatever the plan hands it, which is why creating the AAAA worked fine. But `Records()` had a type filter:

```go
if record.Type != endpoint.RecordTypeA &&
   record.Type != endpoint.RecordTypeCNAME &&
   record.Type != endpoint.RecordTypeTXT {
    continue
}
```

AAAA is not in that list. So the controller could create an AAAA, but when it read the zone back the AAAA was dropped on the floor. The next reconcile saw a desired AAAA with no matching current record, concluded it was missing, and planned another create. Create, forget, create, forget. The cache was not stale and the provider was not flaky; the controller was simply blind to a record type it was perfectly happy to write.

## Confirming it was a real bug, not just one report

The person who filed it said up front they were not familiar with the codebase and might be wrong, so I did not want to lean on the report. Two things settled it.

First, AAAA is managed by default. The default value of `--managed-record-types` is `A, AAAA, CNAME`. So ExternalDNS manages IPv6 records out of the box; this is not an exotic configuration the provider was never meant to handle.

Second, DNSimple was the odd one out. Every other in-tree provider that supports IPv6 reads AAAA back in its own state function. AWS, Azure, Cloudflare, Linode, Pi-hole all handle it. A single provider silently excluding a default-managed type from its read path is an oversight, not a design choice.

## The fix

Add AAAA to the types `Records()` reads back, so the read side matches the write side and the controller can see the record it created:

```go
if record.Type != endpoint.RecordTypeA &&
   record.Type != endpoint.RecordTypeAAAA &&
   record.Type != endpoint.RecordTypeCNAME &&
   record.Type != endpoint.RecordTypeTXT {
    continue
}
```

I added an AAAA record to the provider's existing test fixture so this stays covered. The nice property of that test is that it fails for the right reason before the change: the provider reads back one fewer record than the fixture contains, and the explicit assertion that the AAAA comes back fails too. After the change both pass. A test that does not fail without the fix is not really testing the fix.

## The takeaway

When a controller keeps trying to do something that already succeeded, suspect a mismatch between what it can write and what it can read. A reconcile loop trusts its own read of the world; if that read quietly omits something, the loop will chase it forever. The bug was not in the dramatic-looking apply path, it was in the boring list of record types the read path bothered to return.

The change is in [kubernetes-sigs/external-dns#6517](https://github.com/kubernetes-sigs/external-dns/pull/6517).

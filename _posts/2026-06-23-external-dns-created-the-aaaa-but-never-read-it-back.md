---
layout: post
title: "ExternalDNS created the AAAA record, then forgot it existed"
date: 2026-06-23
author: Miguel Santos
tags: [external-dns]
pr_status: merged
---

ExternalDNS is a reconcile loop. Every minute it reads the records that exist at your DNS provider, reads the records your Kubernetes resources say should exist, diffs the two, and applies the difference. That loop only converges if the read side and the write side agree on what a record is. If the controller can create something it cannot later see, it will create it forever. That is exactly what was happening to AAAA records on the DNSimple provider. The fix looked like one line. Review showed me it was not, because making the record visible turned out to expose a second bug that had been hiding behind the first.

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

The smallest change is to add AAAA to the types `Records()` reads back, so the read side matches the write side. But the provider was hand-maintaining its own list of record types when there is already a shared helper, `provider.SupportedRecordType()`, that most other providers use. So instead of patching the hand-rolled list I switched to the helper:

```go
if !provider.SupportedRecordType(record.Type) {
    continue
}
```

That fixes AAAA and keeps the read path in sync with the rest of the project. It does have a side effect: `SupportedRecordType()` is broader than the old list, so it also lets SRV and NS through. A reviewer asked, reasonably, whether that widens the scope too far. It does not bite in practice, because the controller filters by `--managed-record-types` (default `A, AAAA, CNAME`) before anything reaches the plan. A type the operator has not opted into managing is read and then ignored. The exact same argument is what makes the AAAA fix safe, so the two changes stand or fall together.

## The one-line fix that wasn't

This is the part I would have missed on my own. A maintainer reviewed the change and pointed out that making AAAA visible reaches a second bug that had been dormant precisely because AAAA was invisible.

The delete and update paths do not act on a record directly. They first call `GetRecordID` to look up the provider's internal ID for a record, then delete or update by that ID. And `GetRecordID` matched on name alone:

```go
for _, record := range records.Data {
    if record.Name == recordName {
        return record.ID, nil
    }
}
```

While AAAA records were dropped on read, the controller never planned a delete or update for one, so this lookup was never asked to disambiguate. The moment `Records()` started returning AAAA, that changed. A dual-stack host has an A and an AAAA record at the same name. Ask `GetRecordID` for "www" and it returns whichever of the two it sees first, regardless of type. ExternalDNS would then delete or update the wrong record, silently, on an IPv6 host that finally worked. My one-line fix had quietly armed that.

The fix is to pass the record type through and match on name and type together. DNSimple's list API already accepts a type filter, so the narrowing happens at the API too:

```go
func (p *dnsimpleProvider) GetRecordID(ctx context.Context, zone, recordName, recordType string) (int64, error) {
    listOptions := &dnsimple.ZoneRecordListOptions{Name: &recordName, Type: &recordType}
    // ...
    if record.Name == recordName && record.Type == recordType {
        return record.ID, nil
    }
}
```

## A test that asserted nothing

The same review caught something in the tests I had taken for granted. The existing `ApplyChanges` test only checked that no error came back. It never verified which provider API calls were actually made, and it never called `mockDNS.AssertExpectations(t)`, so the mock's expectations were never enforced. A version of `ApplyChanges` that did nothing at all would have passed it.

I rewrote it as a table-driven test with a fresh mock per case that asserts the exact calls: create per type, the apex domain where the name is stripped to the empty string, update and delete that look up the ID by name and type, and a dual-stack delete where the same name has to resolve to a different ID for A and for AAAA. To prove the dual-stack case earns its place, I reverted the `GetRecordID` fix and watched that one case fail, then restored it. A test that does not fail without the fix is not really testing the fix. I gave the read-path test the same treatment, so it covers all the supported types, dual-stack, the apex, and error propagation instead of spot-checking one record.

## The takeaway

When a controller keeps trying to do something that already succeeded, suspect a mismatch between what it can write and what it can read. A reconcile loop trusts its own read of the world; if that read quietly omits something, the loop will chase it forever.

But the lesson I actually took from this one is about the second bug. Making a record type visible was not a self-contained one-liner; it changed which code paths the record now flows through, and one of those paths had a latent assumption (name is unique) that was only ever true because of the bug I was fixing. Fixes have a blast radius. I would not have seen this one without a careful reviewer, which is most of the argument for sending the change upstream in the open rather than just patching my own fork.

The change is in [kubernetes-sigs/external-dns#6517](https://github.com/kubernetes-sigs/external-dns/pull/6517), now merged.

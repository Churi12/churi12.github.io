---
layout: post
title: "Argo CD could not lock the ref, so it gave up"
date: 2026-06-27
author: Miguel Santos
tags: [argo-cd]
pr_status: merged
---

A maintainer filed an Argo CD bug that caught my eye because it lived right at the seam between a distributed system and a plain old git command. The source hydrator was failing with:

```
 ! [remote rejected]     refs/notes/hydrator.metadata -> refs/notes/hydrator.metadata (cannot lock ref 'refs/notes/hydrator.metadata': is at 4771346... but expected 30b2c90...)
error: failed to push some refs to '...'
```

To reproduce it you run two apps pointing at different destinations, handled by different application controller shards. Two shards, both hydrating, both pushing git notes to the same ref. That is a race, and git tells you so. The interesting part is not that the race happens, it is what the code did when it lost.

## Notes pushes already had a retry

The hydrator stores metadata as git notes under `refs/notes/hydrator.metadata`, and `AddAndPushNote` in the git client already knows these pushes can collide. It runs the push inside a retry loop: fetch the latest notes, add the note locally, push, and if the push fails for a reason that looks like a concurrent update, fetch and try again with exponential backoff.

So someone had already thought about the race. The question was why this particular failure was not being retried.

## The retry only recognised four phrasings

The loop decides whether to retry by string-matching the error:

```go
errStr := err.Error()
isRetryable := strings.Contains(errStr, "fetch first") ||
    strings.Contains(errStr, "reference already exists") ||
    strings.Contains(errStr, "incorrect old value") ||
    strings.Contains(errStr, "failed to update ref")

if !isRetryable {
    return struct{}{}, backoff.Permanent(...)
}
```

Four substrings, each a real way git can report a ref that moved under you. But the message from the bug report does not contain any of them. The concurrent push held the server-side lock, so git said `cannot lock ref`, which is a fifth phrasing nobody had added. The check returned false, the error got wrapped in `backoff.Permanent`, and the retry loop it was sitting inside refused to run. A failure the loop was built to absorb fell straight through it.

This is the trap with classifying errors by substring: the list is only as complete as the failures you have seen. Add a server that phrases the collision differently and the safety net has a hole in it, in exactly the spot it was meant to cover.

## The fix is one more phrasing, and not three

I pulled the check into a small helper and added `cannot lock ref` to the set:

```go
func isRetryableNotePushError(errStr string) bool {
    return strings.Contains(errStr, "fetch first") ||
        strings.Contains(errStr, "reference already exists") ||
        strings.Contains(errStr, "incorrect old value") ||
        strings.Contains(errStr, "failed to update ref") ||
        strings.Contains(errStr, "cannot lock ref")
}
```

The tempting move is to also catch `remote rejected` and `failed to push some refs`, because both show up in that same error block. I left them out on purpose. Those two are generic wrappers: git prints them for protected branches, pre-receive hook denials, and other things that will never succeed no matter how many times you retry. Retrying on them would burn the whole backoff window on pushes that are permanently dead, and worse, it would hide the real reason behind a timeout. `cannot lock ref` is the specific signal that another push is holding the lock right now, which is the one case where trying again is the correct thing to do. Matching the precision of the four entries already there mattered more than catching every line in the message.

## Proving it before trusting it

The classifier is pure string logic, so it is cheap to test directly and there is no excuse not to. I wrote a table-driven test with all four existing signals, a non-retryable auth error that must stay false, and the real `cannot lock ref` block copied from the issue. Then I did the step I never skip: I removed the new line and ran the test. The `cannot lock ref` case failed, returning false where it should return true, which is the bug exactly as reported. Put the line back and it passes. A test you have never watched fail is not guarding anything.

## The takeaway

Retry logic that classifies failures by matching error strings is really a list of the failures someone happened to know about. It looks complete until a different git version or a different server phrases the same race a new way, and then the net has a hole precisely where you needed it. When you add an entry, it is worth asking the opposite question too: which lines look retryable but are actually permanent, so you do not paper over real errors while fixing the transient one. The change is in [argoproj/argo-cd#28469](https://github.com/argoproj/argo-cd/pull/28469), now merged.

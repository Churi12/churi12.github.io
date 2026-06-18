---
layout: post
title: "A timestamp was quietly busting my Go build cache in Alloy"
date: 2026-06-18
author: Miguel Santos
tags: [alloy]
---

Every time I rebuilt Grafana Alloy locally, it relinked the whole binary even when I had changed nothing. For a project this size that is a real chunk of wall-clock on every iteration, and it should not happen — Go's build cache is supposed to make a no-op rebuild nearly free. The cause turned out to be a single linker flag, and the fix is small enough to explain in one post.

## What the build cache promises

Go caches build actions keyed on their inputs: source, flags, dependencies. Run `go build` twice with the same inputs and the second one is a cache hit, including the link step. Change an input and only the affected actions rerun. That is what makes an edit-rebuild loop fast, and it is why a rebuild that relinks every single time is a smell — something is changing an input behind your back.

## The input that kept changing

Alloy, like most Go services, stamps build metadata into the binary with linker flags so `alloy --version` can report it:

```
-X .../build.Version=...
-X .../build.Revision=...
-X .../build.BuildDate=...
```

`-X` sets a string variable at link time. Look closely at `BuildDate`: it is the current time, computed fresh on every invocation. So every build feeds the linker a different value, the link action's inputs never match the cache, and Go faithfully relinks. The cache was working perfectly — it was being told, correctly, that an input had changed. The input just happened to be a clock.

I confirmed it rather than assumed it. Two builds with the timestamp held constant: the second is a cache hit, zero link actions. Two builds with the timestamp changing: the second relinks. That is the whole bug, reproduced in a couple of `go build -x` runs.

## The fix: only stamp the date when the date matters

A build date is useful in a release artifact. It is useless in a throwaway local binary you are going to overwrite in thirty seconds. So the value only needs to be computed when it is actually wanted:

- For official builds (the Makefile already has a `RELEASE_BUILD` switch), stamp the current time as before.
- For reproducible builds, honor `SOURCE_DATE_EPOCH` and stamp that fixed value — unchanged behavior.
- For everything else, the ordinary local dev build, leave it empty.

An empty `BuildDate` is harmless: it is a plain string passed through to the version package, so `alloy --version` just shows an empty date on a dev build. Nothing parses it, nothing panics. In exchange, the link inputs stop changing and repeated local builds go back to being cache hits. Release and reproducible builds produce byte-for-byte what they did before.

## The takeaway

If a build relinks when nothing changed, look for an input that varies on its own: a timestamp, a hostname, a random seed baked in through `-ldflags`. The cache is rarely broken; it is usually being handed a moving target. The reproducible-builds people have known this for years, which is exactly why `SOURCE_DATE_EPOCH` exists — pin the clock and the output stops moving. The same instinct fixes the local loop: do not stamp a clock into an artifact that does not need one.

Small change, but it is the kind I like — read the source, prove the cause, remove the moving part, leave the real builds untouched. The change is in [grafana/alloy#6541](https://github.com/grafana/alloy/pull/6541).

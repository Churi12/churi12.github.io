---
layout: post
title: "What it takes to integration-test a Mimir compactor (and why I trust the test)"
date: 2026-06-18
author: Miguel Santos
tags: [mimir]
---

I added an integration test to Grafana Mimir's compactor recently, and the interesting part was not the test itself — it was how much the project's test harness forces you to verify behavior against the real thing rather than a mock. If you operate Mimir, the way its integration tests are built is worth understanding, because it tells you exactly what guarantees the compactor actually makes.

## The compactor's job, restated as a test

The compactor takes multiple TSDB blocks in object storage and merges them into fewer, larger ones. There are two flavors, and they are easy to conflate:

- **Horizontal compaction** merges blocks covering *adjacent* time ranges into one longer block.
- **Vertical compaction** merges blocks covering the *same* time range that contain the same series — deduplicating and stitching their samples together.

The existing test covered the horizontal case with native histograms. The gap I filled was vertical: take several blocks that all hold the same series within one compaction window, run the compactor, and assert that what comes out is a single block containing the union of every sample. If you operate Mimir, that assertion *is* the guarantee you rely on every time ingesters hand overlapping blocks to the compactor.

## Why these tests run real containers

Mimir's integration tests do not mock the object store or fake the compactor. They use a small framework that spins up actual containers — Consul, MinIO (S3-compatible), and a real Mimir compactor built from your branch — wires them on a Docker network, drives them, and tears them down per test. They are gated behind a build tag so they do not run with normal unit tests:

```go
//go:build requires_docker
```

The flow of a single test is: generate real TSDB blocks from a spec, upload them to MinIO, start the compactor pointed at that bucket, wait on a metric (`cortex_compactor_runs_completed_total`) to know it actually ran, then download the result blocks and read the series back out of the index and chunks to compare against what you expect.

That is heavier than a unit test, and deliberately so. The compactor's correctness depends on real object-store semantics, real block formats, and real index encoding. A mock that returns what you told it to proves nothing about whether the compactor and the storage layer actually agree.

## The part that made me trust the test

Here is the thing I want operators to take away: I did not trust the test until I had watched it fail for the right reason and pass for the right reason.

When I first ran it, it failed in 2.5 seconds — far too fast to have compacted anything. The error was a MinIO port-mapping race in the harness, not my logic. I cleaned up stale containers, re-ran, and it passed in ~50 seconds (the time it takes to actually pull images, boot the compactor, and let a compaction cycle complete). A test that passes instantly is not testing what you think it is. The runtime itself is a signal.

I also got review feedback that the sample comparison was order-sensitive — the compactor may legally return the merged samples across chunks in an order my naive assertion did not expect, which would make the test flaky in CI even though the behavior was correct. So I sorted both sides by timestamp before comparing. A flaky integration test is worse than no test, because it trains people to ignore red.

## What this means if you run Mimir

Two practical takeaways from living in this code:

- **The compactor's vertical-merge guarantee is real and tested:** overlapping blocks with the same series collapse to one block with all samples preserved. If you see duplicate or missing samples after compaction, that is a bug worth reporting, not expected fuzziness.
- **If you ever hack on Mimir,** the integration suite is the fastest way to learn how a component actually behaves, because every test is a runnable, real-container description of a guarantee. Read `integration/compactor_test.go` before you read the compactor source; the tests are the spec.

The test is small — a hundred-odd lines — but writing it meant building the Mimir image locally, running real containers, and watching the thing go green for an honest reason. That is the difference between code that compiles and code I would put my name on.

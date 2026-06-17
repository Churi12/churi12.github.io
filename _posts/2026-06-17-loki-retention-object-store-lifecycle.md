---
layout: post
title: "Loki retention and object store lifecycle policies: the rule that quietly corrupts your bucket"
date: 2026-06-17
author: Miguel Santos
tags: [loki]
---

If you run Grafana Loki on object storage, there is a sentence in the retention docs that is easy to read past and expensive to get wrong:

> Logs are only deleted once the object store lifecycle policy deletes them.

That is true. What the docs did not say, until recently, is *which objects you are allowed to expire* and which ones will quietly break your cluster if you do. I went looking for the answer while setting up retention on S3, could not find it, and ended up reading the Loki source to work it out. This post is what I wish the docs had told me, and most of it is now in the docs because I sent the change upstream.

## The setup that gets you into trouble

You enable the compactor with retention:

```yaml
limits_config:
  retention_period: 672h # 28 days

compactor:
  working_directory: /data/retention
  retention_enabled: true
```

Then someone, reasonably, decides the bucket should also have a lifecycle policy as a cost safety net. The obvious-looking rule is "delete everything older than N days." On S3 that looks like this:

```json
{
  "Rules": [
    {
      "ID": "delete-old-stuff",
      "Status": "Enabled",
      "Filter": { "Prefix": "" },
      "Expiration": { "Days": 30 }
    }
  ]
}
```

An empty prefix means the whole bucket. That rule does not just trim old chunks. It will, given enough time, delete objects that Loki depends on to function, and the failure shows up later as query errors that look nothing like "your lifecycle rule ate the index."

## What actually lives in the bucket

Loki writes more than chunks. The objects fall into two groups, and only one is safe to age out:

| Object or prefix | What it is | Safe to expire? |
| --- | --- | --- |
| `<tenant>/` chunk objects | The log data itself | Yes. This is the only thing a lifecycle rule should target. |
| Index objects under `index/` (the path prefix, `index/` by default) | TSDB/BoltDB index files | No. Loki manages these. Delete them and you remove the references to live chunks. |
| `loki_cluster_seed.json` | A tiny file at the bucket root, written once for usage reporting | No. An age-based rule deletes it the moment it crosses the TTL. |
| Compactor delete-request store | The compactor's record of pending deletions | No. Expire it and the compactor loses track of what it was deleting. |

The one that bit me conceptually was the index prefix. In `schema_config` you set `index.prefix` to something like `loki_index_`, and it is natural to assume that is what you match in a lifecycle rule. It is not. `index.prefix` is the *table name* prefix used *inside* the index path. The actual object key prefix in the bucket is `index/` (the `index.path_prefix` value, which defaults to `index/`). A lifecycle rule matches object key prefixes, so `index/` is what you would accidentally catch, and it is exactly what you must not delete. A reviewer on my docs PR pointed this out and they were right; I had conflated the two until I checked the source.

## The scoped rule that is actually safe

Target only the per-tenant chunk prefix, and keep the expiration longer than your retention period:

```json
{
  "Rules": [
    {
      "ID": "loki-chunk-expiration",
      "Status": "Enabled",
      "Filter": { "Prefix": "fake/" },
      "Expiration": { "Days": 395 }
    }
  ]
}
```

`fake/` is the single-tenant default; with multi-tenancy you use one rule per tenant (or your org ID prefix). The GCS equivalent is the same idea with `matchesPrefix`.

## Two mechanisms that should not fight

The thing that finally made it click: compactor retention and an object store lifecycle policy are two independent deletion mechanisms, and they have very different amounts of knowledge.

- The **compactor is index-aware.** It only deletes a chunk after it has removed every reference to that chunk from the index, and it waits `compactor.retention-delete-delay` (default `2h`) before the sweeper actually removes the marked chunk, so index gateways pick up the rewritten index first.
- An **object store lifecycle policy is index-unaware.** It deletes by age and prefix. It has no idea whether a chunk is still referenced.

For most deployments, the compactor alone is enough and you do not need a lifecycle rule at all. If you add one anyway, set its expiration longer than `retention_period + retention-delete-delay`, so the compactor always deletes first and the lifecycle rule only ever catches objects the compactor already orphaned. That way the two never disagree about a chunk that is still live.

## Takeaways

- Never point a lifecycle rule at the whole bucket. Scope it to the tenant chunk prefix.
- The index lives under `index/`, not under your `index.prefix` table name. Do not expire it.
- `loki_cluster_seed.json` and the delete-request store are not log data and must survive.
- If you must have a lifecycle rule, make its TTL longer than retention plus the delete delay.

Most of this is now in the Loki retention docs. The fastest way I have found to actually understand a tool is to fix the part of its documentation that confused me, because you cannot write the fix until you have read enough of the source to be sure.

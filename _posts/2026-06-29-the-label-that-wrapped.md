---
layout: post
title: "The label that wrapped"
date: 2026-06-29
author: Miguel Santos
tags: [tempo]
---

Someone configured the Tempo data source to show `resource.k8s.cluster_name` and `resource.k8s.namespace` as static search fields in Explore, and the labels overflowed their column. The screenshot in the issue is the whole bug: a tidy form with two field labels that are too long for the space they were given, wrapping and colliding with the inputs next to them. The ask was simple and reasonable — let me rename these. It turned out the reason you could not rename them is the same reason they were too long in the first place.

## Where the label comes from

Static search fields do not store a label. They store a tag, and the label is computed from the tag every time the editor renders. That computation lives in `filterTitle`:

```ts
export const filterTitle = (f: TraceqlFilter, lp: TempoLanguageProvider) => {
  // Special case for the intrinsic "name"
  if (f.tag === 'name') {
    return 'Span Name';
  }
  // Special case for the resource service name
  if (f.tag === 'service.name' && f.scope === TraceqlSearchScope.Resource) {
    return 'Service Name';
  }
  return startCase(filterScopedTag(f, lp));
};
```

The last line is the general case, and it is where the length comes from. `filterScopedTag` prepends the scope, so the tag becomes `resource.k8s.cluster.name`, and then lodash `startCase` runs over it. I had a guess about what that produces and I was wrong, so I ran it:

```
startCase('resource.k8s.cluster.name')  // "Resource K 8 S Cluster Name"
```

`startCase` splits on the dots and on the digit boundary inside `k8s`, capitalizes each piece, and joins with spaces. So a perfectly normal Kubernetes attribute renders as `Resource K 8 S Cluster Name`. That string is long, and it is also slightly mangled — there is no label you could choose for a k8s attribute that `startCase` would not stretch out.

## Where the length hurts

That title is handed to an `InlineField` with a fixed label width:

```tsx
<InlineField label={label} labelWidth={28} grow tooltip={tooltip}>
```

`labelWidth={28}` is a fixed column. A short label fits; `Resource K 8 S Cluster Name` does not, so it wraps, and once it wraps it pushes into the row below. The form has no idea the label is auto-generated and possibly too long — it just lays out what it is given.

So there are two real problems stacked on top of each other: the label is derived rather than chosen, and the column it goes into cannot flex. The issue asked to fix the first one.

## The fix: let the field carry a name

The change is to give a static field an optional label and prefer it when it is set. One field on the filter model:

```ts
export interface TraceqlFilter {
  // ...
  /**
   * A custom label to display for the filter in the search editor.
   * When set, it overrides the auto-generated title derived from the tag.
   */
  label?: string;
}
```

One early return at the top of `filterTitle`, before any of the special cases:

```ts
if (f.label) {
  return f.label;
}
```

And an input on the data source configuration page where you define these static fields, so you can actually type the name in. That input is gated behind a `showLabel` prop threaded through the components, so it shows up where you configure the fields and not in the query editor itself. The query editor needs no changes at all: it already renders through `filterTitle`, so the moment a label exists, the editor picks it up.

The whole thing is additive. A field with no label behaves exactly as before — same `startCase`, same special cases for `name` and `service.name`. Nothing in the stored query changes. You opt in per field, or you ignore it and keep the old behavior.

## On the other fix

There is an earlier comment on the issue suggesting the real fix is to drop the fixed label width, or let the row grow so wrapping does not overflow. That is a good idea and I said so in the PR. It is a CSS fix to the second problem — the column that cannot flex — and it would help every label, including the auto-generated ones, without anyone having to type anything. It is complementary, not competing.

I went with the rename because it is what the issue asked for, and because a later commenter confirmed a custom label would solve their case. A flexible column makes a long name fit; a custom label lets you decide the name should not be long in the first place. `Resource K 8 S Cluster Name` becoming `Cluster` is a better outcome than `Resource K 8 S Cluster Name` fitting more gracefully. Both can ship; neither blocks the other.

## Checking the derivation, not just the override

The tests cover the new path — a label wins, and it wins even over the `name` special case — but the one I cared most about is the fallback, because that is the assertion I had wrong in my head. The test pins `filterTitle` with no label to `Resource K 8 S Cluster Name`, the actual `startCase` output, not the cleaned-up version I assumed. If someone changes how titles are derived, that test fails and tells them the visible label changed. Writing down the ugly real string is more useful than writing down the pretty wrong one.

The change is in [grafana/grafana-tempo-datasource#205](https://github.com/grafana/grafana-tempo-datasource/pull/205).

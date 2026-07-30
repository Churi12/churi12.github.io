---
layout: post
title: "Two green PRs, one broken master"
date: 2026-07-30
author: Miguel Santos
tags: [argo-cd]
pr_status: merged
---

My Argo CD fix merged at 07:40 UTC. By the time I looked at the repo again, master was red, and the failing test was in the file I had touched. Neither PR involved was wrong. That is what made it interesting.

## What landed, and in what order

Two changes went into Argo CD that morning, eighteen minutes apart, both approved, both green:

- **07:22 UTC, [#28518](https://github.com/argoproj/argo-cd/pull/28518)** taught `Repository.Normalize()` to infer the `oci` type from an `oci://` URL. Before that change, `Normalize` only ever filled in the default type, which is `git`.
- **07:40 UTC, [#28904](https://github.com/argoproj/argo-cd/pull/28904)** was mine. It fixed an untyped Helm chart source being resolved as a git repository, and the way it fixed it was to stop calling `Normalize()` at the top of `UpdateRevisionForPaths` and decide git-ness from the raw request instead.

Read those two sentences together and the collision is obvious. One PR made `Normalize` the thing that knows how to type an OCI repository. The other removed the call to `Normalize`.

Neither CI run could have caught it. #28518 was tested against a tree where my change did not exist. Mine was tested against a tree where the OCI inference did not exist. Both were honestly green. The merge produced a state that no CI run had ever evaluated.

## The panic

The failing case was `OCIRepoWithEmptyTypeShortCircuits`, a test added by #28518. It passes a repository with an `oci://` URL and **no** type, and no `ApplicationSource` at all. The point of the test is that such a repo should short-circuit and never touch the git path.

With `Normalize()` gone, the empty type stayed empty. My fallback logic then had nothing to fall back to, because `ApplicationSource` was nil, so it resolved to git. The function walked into `gitSourceHasChanges`, which called `TempPaths.GetPath("oci://example.com/foo")` on a mock that had no expectation for that argument, and the test panicked rather than failed.

## Fixing it without reintroducing the original bug

The naive repair is to put `Normalize()` back. That fixes the OCI test and restores the Helm bug I had just fixed, because after `Normalize` an untyped Helm source reads as `git` again.

The thing that makes both work is remembering one bit of information before it gets destroyed:

```go
// Remember whether the type was set by the caller before normalizing: Normalize
// infers oci from an oci:// URL and otherwise defaults the type to git, so after the
// call an empty type is indistinguishable from an explicit git type.
typeWasUnset := repo.Type == ""
repo = repo.Normalize()

isGitSource := repo.Type == "git"
if isGitSource && typeWasUnset && request.ApplicationSource != nil &&
    (request.ApplicationSource.IsHelm() || request.ApplicationSource.IsOCI()) {
    isGitSource = false
}
```

`Normalize` runs, so OCI inference works and every other field is populated as callers expect. But a `git` type that `Normalize` invented is not trusted on its own. It only gets overridden when the caller left the type blank **and** the application source says Helm or OCI. An explicitly typed git repository keeps the git path no matter what the source looks like, and the OCI test short-circuits because `Normalize` now types it as `oci`.

The `typeWasUnset` guard is the whole fix, so I made sure it was actually tested. I added `ExplicitGitTypeWithHelmSourceStillTreatedAsGit`: a repo with `Type: "git"` set explicitly, an application source with a chart, and an expectation that the revision still resolves. Then I deleted the guard locally and confirmed the new test fails. A guard nothing tests is a guard that will be removed by the next person cleaning up.

## The review comment after the merge

The fix merged the same day. A few minutes later a reviewer asked whether the new test could assert the `Changes` field as well, since it only listed a revision.

It turned out `Changes` was already asserted, just implicitly. The table runner compares the entire response struct:

```go
assert.Equalf(t, tt.want, got, "UpdateRevisionForPaths(%v, %v)", tt.args.ctx, tt.args.request)
```

Because `want` omitted `Changes`, it took the zero value `false`, and the struct comparison enforced it. I checked that the assertion has teeth by flipping the expectation to `Changes: true` locally, which fails as expected. So there was no coverage gap.

The point still stood on readability, though. `Changes: false` is the outcome that proves the git path ran and found no changes, and leaving it implicit means a reader cannot tell whether `false` was intended or simply forgotten. The sibling case directly above it already spells it out. So I opened a one-line follow-up, [#28984](https://github.com/argoproj/argo-cd/pull/28984), rather than leaving the reply as an argument for why I was already right. Being technically correct and being clear are different goals, and the reviewer was asking for the second one.

## The takeaway

CI proves your change is correct against the tree it ran on. On a busy repository, that tree may not exist by the time you merge, and the gap widens with every day your PR waits for review. Mine had been approved and sitting for a while, which is exactly the window this failure mode needs.

The habit I have added: before a long-pending PR merges, re-run the affected package against fresh master, not against my branch point. And when the interaction is with a shared helper like `Normalize`, check who else has been changing it. In this case a quick `git log` on that one file would have shown #28518 landing an hour earlier and touching the exact function I was removing a call to.

The repair is in [argoproj/argo-cd#28979](https://github.com/argoproj/argo-cd/pull/28979), merged the same day it broke.

---
layout: post
title: "Argo CD resolved my Helm chart as a git repo"
date: 2026-07-26
author: Miguel Santos
tags: [argo-cd]
pr_status: merged
---

Someone reported an Argo CD regression that only shows up in a fairly specific shape of Application: a multi-source app that combines an untyped Helm chart source with a git values source and the `argocd.argoproj.io/manifest-generate-paths` annotation. On v3.4.3 it started failing with:

```
unable to resolve git revision : failed to list refs: repository not found
```

The chart repository is not a git repo, so of course listing git refs against it fails. The real question is why Argo CD was trying to list git refs against a Helm chart URL at all. It had all the information it needed to know this was a Helm source.

## The guard that used to short-circuit

`UpdateRevisionForPaths` in the reposerver decides, per source, whether it needs to resolve a git revision for the manifest-generate-paths optimization. A non-git source with no ref sources has nothing to resolve, so the function returns early:

```go
if repo.Type != "git" && len(request.RefSources) == 0 {
    return &apiclient.UpdateRevisionForPathsResponse{}, nil
}
```

And the git work itself is gated the same way:

```go
if repo.Type == "git" {
    // resolve the git revision...
}
```

For an untyped Helm chart source with ref sources, this is exactly the seam where things go wrong.

## Normalize turned "unset" into "git"

The regression came in with one added line. Someone put a `repo.Normalize()` call at the top of the function to tidy up the request:

```go
repo = repo.Normalize()
```

`Normalize` is helpful in most places, but it does one thing that matters here: when the repository type is unset, it coerces it to the default, which is `git`. An untyped Helm chart source (no explicit type, chart repo not registered as a repository secret) has an empty type. After `Normalize`, that empty type is `git`.

Now walk the two guards again. The source has ref sources, so the early return does not fire. Its type is now `git`, so it enters the git branch and tries to resolve a git revision against the Helm chart URL. `repository not found`. Before `Normalize` was added, the empty type was not `git`, the early return fired, and the function did the right thing by doing nothing.

The bug is not in `Normalize`. It is in treating "unset" as "definitely git" at a decision point where unset could just as easily mean Helm or OCI.

## The fix: decide git-ness before normalizing

Instead of letting `Normalize` guess, I capture whether the source is really git before it runs, and fall back to the application source type when the repository type is genuinely unset:

```go
isGitSource := repo.Type == "git"
if repo.Type == "" {
    isGitSource = request.ApplicationSource == nil ||
        (!request.ApplicationSource.IsHelm() && !request.ApplicationSource.IsOCI())
}
```

Then both guards key off `isGitSource` instead of `repo.Type`. An untyped source is only treated as git when the application source is not Helm and not OCI. A genuine git repo that happens to use Helm value overrides (a git source with `Helm:` set but no `Chart:`) stays on the git path, because `IsHelm` keys on the chart, not on the presence of Helm value config. That distinction is the whole reason the fix is safe.

## Proving it before trusting it

The test is the part I never skip. I added a case to `TestUpdateRevisionForPaths` for an untyped Helm chart source carrying ref sources, and wired the git client mock so that any git resolution returns `failed to list refs: repository not found`. That means the test only passes if the git path is never entered. On master the new case fails with the exact error from the issue; with the fix it passes and does zero external cache work. A test that reproduces the reported error message before the fix, and goes green after, is the one I trust.

## An aside on trusting autofixes

While iterating on this PR I accepted a Copilot autofix suggestion in the GitHub web UI, and it committed a dead-assignment lint error (`SA4006`) straight into the branch, plus a bit of working-tree contamination that leaked unrelated notification files into the diff. Accepting a web autofix commits it as-is, with no local gofmt, lint, or build in front of it. I caught both and cleaned them up, but the lesson stuck: an autofix is a suggestion, not a verified change. Run it through the same gate as anything you write by hand.

## The takeaway

`Normalize`-style helpers that fill in defaults are convenient right up to the moment a downstream decision depends on the difference between "unset" and "the default value." Coercing empty-to-git erased exactly the signal the guard needed. When a default is applied, ask whether anything later treats the defaulted value as if the user had chosen it, because that is where the surprises live. The change is in [argoproj/argo-cd#28904](https://github.com/argoproj/argo-cd/pull/28904), now merged. It also had a sequel: it landed eighteen minutes after another PR it silently conflicted with, and broke master. That story is in [Two green PRs, one broken master](/2026/07/two-green-prs-one-broken-master/).

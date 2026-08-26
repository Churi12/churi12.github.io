---
layout: post
title: "The cache was right there, just not reachable"
date: 2026-07-26
author: Miguel Santos
tags: [argo-cd]
---

Argo CD's notification controller had a performance regression in v3.4.x: with a lot of applications and triggers, it was making a large number of API calls per reconcile. The report traced it to AppProject resolution during template evaluation. Every trigger evaluation that needed the `appProject` template variable did a live API `Get` for the project. Multiply that by applications times triggers and the API server feels it.

What made this one interesting is that the fix was not "add a cache." The cache already existed. It just could not be reached from the code that needed it.

## Two ways to fetch the same AppProject

The notification controller already runs a synced `AppProject` informer. There is even a helper, `getAppProj`, that reads projects straight out of that informer's indexer. No API call, just a local cache lookup on an already-synced store.

But the template variables are built in a different package, `util/notification/settings`, by `getAppProjectForTemplate`. That function had no access to the informer, so it did the only thing it could:

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

appProjectObj, err := argocdService.GetAppProject(ctx, projectName, namespace)
```

A live `Get`, on every trigger evaluation. The fast path existed a package away; the settings package simply had no handle to it.

## Threading the cache in without forcing it on everyone

The fix is to give the settings package an optional way to read from a cache, and let the controller supply one backed by its existing informer. I added a small function type:

```go
// AppProjectGetter retrieves an AppProject from a local cache, avoiding a live
// API call on every trigger evaluation. It returns a nil object without an
// error when the AppProject is not present in the cache.
type AppProjectGetter func(namespace, name string) (*unstructured.Unstructured, error)
```

`getAppProjectForTemplate` prefers it when present, and only falls back to the live lookup when it is nil:

```go
if appProjectGetter != nil {
    appProjectObj, err := appProjectGetter(namespace, projectName)
    if err != nil {
        log.WithFields(logFields).Warnf("Failed to get AppProject for notification template: %v", err)
        return nil
    }
    if appProjectObj == nil {
        return nil
    }
    return appProjectObj.Object
}

// ... otherwise the previous live Get, unchanged.
```

The notification controller builds a getter that reads the informer's indexer:

```go
func newAppProjectGetter(appProjInformer cache.SharedIndexInformer) settings.AppProjectGetter {
    return func(namespace, name string) (*unstructured.Unstructured, error) {
        projObj, exists, err := appProjInformer.GetIndexer().GetByKey(fmt.Sprintf("%s/%s", namespace, name))
        // ... return the unstructured project, or nil if missing
    }
}
```

Callers that have no local cache, like the API server and the CLI, pass `nil` and keep the exact behavior they had before. So the controller, the hot path, gets the cache; everyone else is untouched.

## The honest note about the signature

`GetFactorySettings` is an exported function, and this change adds an `AppProjectGetter` parameter to it. I updated every in-tree caller, but if some out-of-tree code calls that function, this is a breaking change. I flagged that plainly in the PR body rather than hiding it. It seems like a reasonable cost for a real performance fix, but it is the maintainers' call, and they should get to make it with the fact in front of them, not discover it later.

## Proving it before trusting it

The test I care most about here does not assert a return value, it asserts an absence. `TestGetAppProjectForTemplateUsesCache` provides a getter and a mock `Service` with no `GetAppProject` expectation configured at all. If the code ever fell back to the live lookup, the mock would fail the test. So the test proves the cache path is taken and the API is never touched, which is the whole point of the fix. A second test covers a cache miss returning nil without falling back. I confirmed both by reverting the cache branch and watching them fail.

## The takeaway

Sometimes the fix for "this does too much I/O" is not a new cache or a new client, it is a wire. The right data was already sitting in memory, synced and ready; it just was not plumbed to the caller that needed it. When you see a hot path doing live lookups, check whether something nearby already has the answer cached before you reach for a bigger hammer. The change was in [argoproj/argo-cd#28905](https://github.com/argoproj/argo-cd/pull/28905).

## Postscript: the same fix, under someone else's number

My PR was closed on 24 August, unmerged, with one line: closing in favor of [#28815](https://github.com/argoproj/argo-cd/pull/28815). That is the right call, and it is worth writing down why rather than quietly deleting this post.

[#28815](https://github.com/argoproj/argo-cd/pull/28815) does the same thing I did, reads the AppProject from the informer cache instead of doing a live `Get` per trigger evaluation, and it does it better in two ways. It threads the informer in properly, as a `SetAppProjectInformer` on the notification service wired up in `NewController`, instead of my optional getter function and the exported-signature change I had to apologise for above. And it fixes a second bug in the same code path that I walked straight past: the old lookup used the application's namespace rather than the control-plane namespace, so for anyone running apps-in-any-namespace it 404s and logs a failure for every application on every reconcile. I was so focused on the call count that I never questioned the arguments.

It was also opened on 20 July, six days before mine.

That last fact is the actual lesson, and it is not a technical one. I checked for existing work before starting, the way I always do: I read the issue I was fixing and followed its cross-references. But the two PRs cite different issues, mine [#28530](https://github.com/argoproj/argo-cd/issues/28530) and theirs [#28137](https://github.com/argoproj/argo-cd/issues/28137), so nothing linked them and the timeline check came back clean. Two people had found the same regression from two different reports. What would have caught it is a search I did not run: the files. Both PRs touch `notification_controller/controller/controller.go` and `util/notification/settings/settings.go`, and a plain PR search on either path would have shown me #28815 sitting there before I wrote a line.

So the analysis in this post held up, the fix shipped, and the code I wrote is not in Argo CD. I would rather have spent that week on something nobody else was already doing. Now I search by file before I search by issue.

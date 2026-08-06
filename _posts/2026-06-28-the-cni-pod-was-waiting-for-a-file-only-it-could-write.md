---
layout: post
title: "The CNI pod was waiting for a file only it could write"
date: 2026-06-28
author: Miguel Santos
tags: [istio]
pr_status: merged
---

A node reboots. Every pod that tries to come back up fails to get a network:

```
failed to setup network for sandbox ...: ... istio-cni-kubeconfig: no such file or directory
```

The file that is missing, `istio-cni-kubeconfig`, is written by the istio-cni node agent. So the agent needs to start, write the file, and then everything else can use it. The catch is that the agent is also a pod, and bringing up a pod runs the CNI plugin, which wants to read that same kubeconfig. The thing that creates the file is blocked behind the file it creates. Classic deadlock, and a reboot is the moment it bites because nothing has written the file yet.

## Someone already saw this coming

What is interesting is that the code already had the escape hatch. The CNI plugin's `CmdAdd` does a preemptive check: before it builds a Kubernetes client, it asks "is the pod I am being called for actually my own node agent?" If so, it returns early and never touches the kubeconfig. The comment spells out the reasoning exactly:

```go
// Preemptively check if the pod is a CNI pod.
// It is possible that the kubeconfig is not available if it hasn't been written yet
// by the CNI pod ... to avoid a deadlock on the kubeconfig when the k8s client is
// unnecessary to process the CNI add event for the CNI pod itself.
```

That is the deadlock, named and defended against. The detection is a small helper that checks whether the pod name has the `istio-cni-node-` prefix and lives in the agent's own namespace. If it does, skip the client entirely and let the pod through.

## The guard only ran in one mode

Here is the line that turned a fixed bug back into an open one:

```go
if conf.AmbientEnabled {
    k8sArgs := K8sArgs{}
    ...
    if isCNIPod(conf, &k8sArgs) {
        return pluginResponse(conf)
    }
}
```

The whole preemptive check sits inside `if conf.AmbientEnabled`. So the deadlock guard only fires for clusters running ambient mode. A plain sidecar-mode cluster never runs it, and right after this block the code does what it always does:

```go
client, err := newK8sClient(*conf)
```

`newK8sClient` runs unconditionally, in every mode. It is the call that blocks on the missing kubeconfig. So the guard protects against a deadlock that can happen in both modes, but only in the one mode it happens to be nested under. The reasoning in the comment is not specific to ambient at all, it is about the agent pod needing a client it is not yet able to build. The condition around it was narrower than the problem it described.

## The fix is to stop nesting it

The check has nothing to do with ambient mode, so it should not be gated by it. I lifted it out:

```go
k8sArgs := K8sArgs{}
if err := types.LoadArgs(args.Args, &k8sArgs); err != nil {
    return fmt.Errorf("failed to load args: %v", err)
}
if isCNIPod(conf, &k8sArgs) {
    return pluginResponse(conf)
}
```

Now the agent pod is recognised and skipped before the client is built, regardless of mode. The conditional was load-bearing for ambient only by accident; the behaviour it guards belongs to both.

## Proving it before trusting it, on Linux

The CNI package is full of Linux network-namespace code, so the tests do not even compile on my Mac. I ran them in a `golang:1.26` container instead, which is worth knowing if you ever touch this corner of istio: a green run locally on macOS is not available to you here, you have to get to Linux somehow.

The test calls `CmdAdd` for a pod named like the node agent, in both ambient and sidecar mode, and asserts it returns without error. The mock kubeconfig does not exist, so if the code reaches `newK8sClient` the test fails. I ran it both ways. With the fix, both modes pass. With the guard still nested under `AmbientEnabled`, the ambient subtest passes and the sidecar subtest fails, reaching the client and erroring out, which is the deadlock reproduced in a unit test. The two-mode split in the result is the bug, drawn precisely.

## The review found a second, smaller version of the same mistake

The PR sat for over a month before a maintainer looked at it, and when one did, the note was not about the fix at all. It was about a logging change I had bundled in. `isCNIPod` returning false used to log a warning, and since my new preemptive check calls it on literally every pod add, a warning per pod is noise. So I had lowered it to debug.

He was right to push back, and the reason is worth writing down. Once I lifted the check out, `isCNIPod` had **two** callers with opposite needs. The new preemptive one runs on every add, where a false result is completely unremarkable. The other lives in a degraded failsafe that only runs when the API server is unreachable, and there a false result means we are about to keep blocking that pod. Same function, same return value, two entirely different levels of interesting. Changing the level inside the helper could only be wrong for one of them, and I had picked the one that destroyed the signal that actually mattered.

The fix was to stop making the helper decide. It became a pure predicate that does not log at all, and each caller logs at the level that suits its own situation. That framing also surfaced two things nobody had flagged. The degraded-path warning sat inside a retry loop of `podRetrievalMaxRetries = 30` at one second apart, and the verdict cannot change between attempts, so a single blocked pod produced thirty identical warnings; it is now guarded to fire once. And the message on the true branch said "in a degraded state", which was simply false on the new preemptive path, because that path is the healthy one.

It merged the next day, with cherry-picks queued for release-1.30 and release-1.31.

## The takeaway

A fix that is correct but scoped too narrowly is its own kind of bug, and it is harder to spot than a missing fix because the defensive code is right there, commented, clearly intended. The tell was the mismatch between the comment and the condition: the comment justified the check in mode-neutral terms while the `if` wrapped it in one mode. When a guard's rationale is broader than the branch it lives in, the branch is usually wrong, not the rationale. And the review note is its own smaller version of the same lesson: a helper that logs is making a decision on behalf of callers it cannot see. The change is in [istio/istio#60721](https://github.com/istio/istio/pull/60721), now merged.

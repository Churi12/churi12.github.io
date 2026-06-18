---
layout: post
title: "Karpenter provisions, kube-scheduler binds: why your pod landed somewhere surprising"
date: 2026-06-18
author: Miguel Santos
tags: [karpenter]
---

A pod is pending. Karpenter launches a node for it. The node comes up, the pod schedules, and then a minute later that node is gone again, empty, consolidated away. Or the opposite: you set a zonal topology spread and Karpenter launches three nodes when you expected one. Neither is a bug, but both confuse people, and the confusion almost always comes from the same place: treating Karpenter and the Kubernetes scheduler as one thing when they are two.

I went down this rabbit hole while documenting it upstream, and the mental model that fixed it for me is small enough to fit in a sentence: **Karpenter decides what capacity to launch, `kube-scheduler` decides where pods actually go.** Everything surprising follows from those being separate decisions made by separate components at separate times.

## Two jobs, not one

When a pod cannot fit on existing capacity, Karpenter runs its own internal scheduling simulation. It models your pending pods against your NodePools and the available instance types, and from that it decides what new nodes to create. That is the whole of Karpenter's job here: it is a *provisioning* decision. The output is "launch an `m5.2xlarge` in `eu-west-1a`," not "put pod X on node Y."

The actual binding of pods to nodes is still done by the upstream `kube-scheduler`, exactly as it is on any cluster. Once Karpenter's node boots, registers, and goes `Ready`, the scheduler does its normal filtering and scoring across all nodes and binds the pending pods. Karpenter does not set `nodeName` on your pods and it does not bypass the scheduler.

So a pod's life crosses a boundary: Karpenter gets capacity to exist, then hands off to the scheduler to place the pod. The two agree most of the time. The interesting cases are when they do not.

## Where they diverge

### Preferred constraints are read differently

This is the big one. Karpenter, when it builds the requirements for a pod during simulation, **treats preferred affinities as if they were required**, and only relaxes them one at a time if it cannot otherwise satisfy them. It does this so it can provision capacity that honors your preferences.

`kube-scheduler` does the opposite: a `preferredDuringSchedulingIgnoredDuringExecution` affinity, or a `ScheduleAnyway` topology spread, is folded into its *scoring*. If nothing better is available, the scheduler will happily place the pod somewhere that does not satisfy the preference at all.

The practical fallout: a pod can land on a different node than the one Karpenter sized for it, and provisioning to satisfy a soft preference can launch **more nodes than you expected**. If you set a preferred zonal spread on a deployment and scale it up, Karpenter may launch a node per zone to honor the spread, even though the scheduler would have been content to pack everything onto the first node that became ready.

### A pod can grab existing capacity before the new node joins

Karpenter decides to launch a node from a snapshot of what is pending *right now*. But a node takes time to boot and register, sometimes a couple of minutes. In that window, other things happen: a pod terminates, another node finishes booting, a disruption completes. Suddenly there is room on existing capacity, and `kube-scheduler` binds the pending pod there instead of waiting for Karpenter's new node.

The new node then comes up lightly used or completely empty. This is the "why did Karpenter make a node and then delete it" case. It is working as intended: the scheduler did its job (the pod is running), and Karpenter's consolidation later reclaims the node nobody needed. It looks like thrash; it is actually two correct components racing.

### Bin-packing versus per-pod placement

Karpenter batches pending pods and bin-packs them to pick cost-effective instance types. The node it launches is sized for a *group* of pods. But `kube-scheduler` binds each pod individually, scoring nodes one pod at a time. So the specific node a given pod ends up on is not guaranteed to be the one Karpenter mentally assigned it to during packing. The totals work out; the individual assignments may not match the simulation.

## What to do about it

None of this needs fixing, but it does change how you should express intent:

- **If placement actually matters, use required constraints, not preferred ones.** `requiredDuringSchedulingIgnoredDuringExecution` (under `spec.affinity.nodeAffinity`) and `whenUnsatisfiable: DoNotSchedule` for topology spread. Karpenter and `kube-scheduler` agree far more closely on hard requirements than on soft ones, because there is no "relax it" or "score it lower" path to disagree about.
- **Treat preferred affinities and `ScheduleAnyway` as best-effort, and budget for extra nodes.** If you do not want Karpenter launching capacity to chase a preference, do not encode the preference as something it will try to satisfy.
- **Expect transient nodes during scale-up and disruption.** A node that comes up empty and gets consolidated is not a failure. If it bothers you, look at why pods are landing elsewhere (usually existing capacity freeing up mid-launch), not at Karpenter.
- **If a pod stays pending even though Karpenter launched a node for it,** check that the node's labels, taints, and zone actually satisfy the pod's *required* constraints. Those are what the scheduler enforces at bind time, and a mismatch there is the usual culprit.

The one-line version, again: Karpenter provisions, `kube-scheduler` binds. Hold those two apart in your head and the surprising behavior stops being surprising. Most of this is now in the Karpenter scheduling docs, because the fastest way I have found to actually understand a system is to write down the part of it that confused me.

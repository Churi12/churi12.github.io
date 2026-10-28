---
layout: post
title: "The actions menu that needed a status first"
date: 2026-10-21
author: Miguel Santos
tags: [argo-cd]
---

Argo CD lets you run actions on a resource from the UI: restart a Deployment, resume a paused Rollout, promote a canary. Which actions show up, and which are greyed out, is decided by a small Lua script per resource kind. A user reported that for an Argo Rollout with no `status`, the whole menu failed to load:

```
error getting available actions: <string>:15: attempt to index a non-table object(nil) with key 'currentPodHash'
```

A Rollout has no `status` until the Argo Rollouts controller reconciles it. Right after it is created, while the controller is down, or when no controller handles it, the field is simply not there. That is exactly when you might want to press Restart, and the menu was gone, Restart included.

## The script already knew

The discovery script for `argoproj.io/Rollout` is short, and parts of it were already written for a missing status:

```lua
local paused = false
if obj.status ~= nil and obj.status.pauseConditions ~= nil then
    paused = table.getn(obj.status.pauseConditions) > 0
elseif obj.spec.paused ~= nil then
    paused = obj.spec.paused
end
```

and further down, `promote-full` is only computed under `if obj.status ~= nil`. So whoever wrote it knew status could be nil. Two lines in between did not check:

```lua
local fullyPromoted = obj.status.currentPodHash == obj.status.stableRS
```

and the `skip-current-step` line, which reads `obj.status.currentStepIndex`. One unguarded line is enough. Lua raises on the first index into nil, the script never returns, and Argo CD has no partial result to show.

## What no status means

The fix is not just "add a nil check". It has to answer a question: with no status, which actions make sense? No status means the controller has not looked at the Rollout yet, so no update can be in progress. That is the same answer as "fully promoted":

```lua
-- without a status the controller has not reconciled the Rollout yet, so no update is in progress
local fullyPromoted = obj.status == nil or obj.status.currentPodHash == obj.status.stableRS
```

The nice part is the three lines right after it:

```lua
actions["abort"] = {["disabled"] = fullyPromoted or obj.status.abort}
actions["retry"] = {["disabled"] = fullyPromoted or not(obj.status.abort)}
actions["pause"] = {["disabled"] = fullyPromoted or obj.status.abort or obj.spec.paused }
```

They also read `obj.status`, but Lua's `or` short circuits. With `fullyPromoted` true, the right side is never evaluated, so they are safe without touching them. `skip-current-step` got an explicit `obj.status == nil` in its condition. Restart stays enabled, and Resume still follows `spec.paused`.

## The one I missed

Copilot reviewed the PR and found something I had not looked at. Before the fix, Resume could never be offered on a Rollout without status, because discovery crashed first. After the fix, a paused Rollout without status shows Resume as enabled. And the Resume action itself started like this:

```lua
if obj.status.pauseConditions ~= nil and table.getn(obj.status.pauseConditions) > 0 then
```

So the menu would load, you would press Resume, and that would crash instead. The fix had made a second bug reachable. It got the same guard, `obj.status ~= nil and ...`, and a test that runs discovery and then the Resume action on a paused Rollout without status.

## Tests

Argo CD tests these scripts with YAML fixtures in `action_test.yaml`: an input object and the expected list of actions, or the expected object after an action runs. I added a canary Rollout with steps and no status. It fails on master with the same error as the issue and passes with the fix. Then the paused case for Resume.

## What I took from it

A crash hides everything behind it. While discovery failed on line 15, nobody could reach the Resume action with a nil status, so nobody could find that it failed too. When you fix a crash, the code after it starts running on inputs it has never seen. It is worth reading what becomes reachable, not just the line that threw.

The change is in [argoproj/argo-cd#30331](https://github.com/argoproj/argo-cd/pull/30331).

---
layout: post
title: "A secret got printed as a Go struct into my scrape target"
date: 2026-06-27
author: Miguel Santos
tags: [alloy]
pr_status: merged
---

Someone reported an Alloy config where they read a value out of a file and used it as a scrape target address. Instead of the address, the target came out as `{true 10.0.0.1:9090}` — the IP wrapped in what is unmistakably a Go struct printed with `%v`. The workaround they found was to wrap the value in `convert.nonsensitive()`, which is a strong hint about where to look. This is the kind of bug I like: the symptom names the cause if you know the codebase, and the fix is a few lines once you see it.

## The setup that breaks

The config reads a file and feeds its content into a `discovery.relabel` target:

```
local.file "service_ip" {
  filename  = "/etc/alloy/service_ip"
  is_secret = true
}

discovery.relabel "backend" {
  targets = [{
    __address__ = local.file.service_ip.content,
  }]
}
```

You expect `__address__` to be the file's contents. You get `{true 10.0.0.1:9090}`. Flip `is_secret` to `false` and it becomes `{false 10.0.0.1:9090}`. That `true`/`false` leading the value is the tell.

## Why a string turns into a struct

`local.file` does not export its content as a plain string. It exports an `alloytypes.OptionalSecret`, which is a small struct:

```go
type OptionalSecret struct {
	IsSecret bool
	Value    string
}
```

That is so Alloy can carry "this might be sensitive" alongside the value and redact it when rendering config. It is a capsule type, not a string.

Targets are built in `discovery`'s `Target.ConvertFrom`. When it walks the map of target values, it handles strings and then falls back to formatting anything else:

```go
case v.IsString():
	strValue = v.Text()
case v.Reflect().CanInterface():
	strValue = fmt.Sprintf("%v", v.Reflect().Interface())
```

An `OptionalSecret` is not a string, so it hits the second branch, and `%v` on a struct gives you exactly `{true 10.0.0.1:9090}`. The boolean is `IsSecret`, the rest is `Value`. The code was never wrong about the value — it just printed the whole box instead of what was inside it. And `convert.nonsensitive()` worked around it precisely because it turned the capsule back into a plain string before it reached this branch.

## The fix I wrote: unwrap the box

The two capsule types that wrap a string here are `OptionalSecret` and `Secret`. My first fix pulled the underlying string out of them before the `%v` fallback:

```go
case v.Reflect().CanInterface():
	switch raw := v.Reflect().Interface().(type) {
	case alloytypes.Secret:
		strValue = string(raw)
	case alloytypes.OptionalSecret:
		strValue = raw.Value
	default:
		strValue = fmt.Sprintf("%v", raw)
	}
```

I flagged one wart in the PR and talked myself past it: when the value really is a secret, this puts the real string into the target label. My argument was that it is not a *new* leak, since the old code already put that same string into the label, just wrapped in `{true ...}`. Unmangling it, I reasoned, does not make the exposure worse.

## The maintainer found the assumption under the wart

kalleep did not accept the framing, and he was right not to:

> it think it's an oversight that we even convert secrets here in the first place, even though the value is not usable. IMO we should not convert Secrets at all and only convert optional secrets if `IsSecret == false`.

That reply reframes the whole bug. I had been asking "what string is inside this box?" when the question was "should a secret ever become a target address at all?" And it cannot: a scrape target is a label value, it gets logged, exported in metrics metadata, and shown in the UI. A secret is not merely awkward there, it is *never usable* there. The right answer is not to unwrap it more cleanly, it is to refuse.

So the shipped fix rejects instead of converting:

```go
switch raw := rv.Interface().(type) {
case alloytypes.Secret:
	return fmt.Errorf("target::ConvertFrom: cannot use a secret as a target value")
case alloytypes.OptionalSecret:
	if raw.IsSecret {
		return fmt.Errorf("target::ConvertFrom: cannot use a secret as a target value")
	}
	strValue = raw.Value
default:
	strValue = fmt.Sprintf("%v", raw)
}
```

An `OptionalSecret` holding a non-secret still unwraps to its string, which is the actual reported case and the thing that was broken. A real `Secret`, and an `OptionalSecret` that is genuinely secret, now produce a config error at evaluation time instead of quietly landing in a label.

While I was in there I also handled a nil capsule value, which the old `CanInterface()` branch would have panicked on rather than reported.

I prefer the result to what I proposed. My version made the symptom go away and left the design mistake in place; his makes the config fail loudly at the point where someone asked for something that cannot work.

## Proving it before trusting it

The part I never skip: I wrote the test, then reverted the fix to watch it fail. With the fix gone the non-secret case produces `{false 10.0.0.1:9090}`, the same struct-instead-of-string shape as the bug report, and with the fix back it produces `10.0.0.1:9090`. A test that has never been seen to fail is not a regression guard, it is decoration.

After the rework the tests assert the new contract rather than the old one: a non-secret optional unwraps, a real `Secret` and a secret-holding `OptionalSecret` each produce the specific rejection error, and a nil capsule value errors instead of panicking. Asserting the *specific* error matters — an earlier version of the test accepted any evaluation failure, which would have passed even if the config had broken for an entirely unrelated reason.

## The takeaway

`fmt.Sprintf("%v", x)` is a fine last resort and a bad default. The moment `x` can be a struct that wraps the thing you actually wanted, `%v` will happily hand you the wrapper. When a value passes through a type system that boxes things — secrets, optionals, capsules — the conversion back out is a real step, not something to leave to default formatting.

But the lesson I actually took away is the other one. I noticed the uncomfortable part of my own fix, wrote it down honestly in the PR, and then argued for why it was acceptable. Writing a caveat down is not the same as resolving it. When you catch yourself explaining why a wart is tolerable, that is the moment to ask whether it is pointing at a wrong assumption one level up — because a reviewer will, and they will be reading the design while you are still defending the diff. The change is in [grafana/alloy#6605](https://github.com/grafana/alloy/pull/6605), now merged.

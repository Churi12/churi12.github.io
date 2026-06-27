---
layout: post
title: "A secret got printed as a Go struct into my scrape target"
date: 2026-06-27
author: Miguel Santos
tags: [alloy]
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

## The fix: unwrap the box

The two capsule types that wrap a string here are `OptionalSecret` and `Secret`. The fix is to pull the underlying string out of them before the `%v` fallback:

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

I thought about doing this generically — ask any capsule to convert itself to a string — but `OptionalSecret` deliberately refuses that conversion when it actually holds a secret, so a generic path would either drop the value or fall back to the struct again. Naming the two types keeps the behavior explicit, which is what you want in code that builds scrape targets.

One thing worth being honest about: when the value really is a secret, this now puts the real string into the target label. That is not a new leak. The old code already put that same string into the label — just wrapped in `{true ...}`. If you put a secret in a target, it ends up in the target either way; this change only stops mangling it. Putting a secret in a target address is arguably the wrong move, but that is a separate conversation from "the value should not be a struct literal."

## Proving it before trusting it

The part I never skip: I wrote the test, then reverted the fix to watch it fail. With the fix gone the test produces `{true 10.0.0.1:9090}` — the exact string from the bug report — and with the fix back it produces `10.0.0.1:9090`. A test that has never been seen to fail is not a regression guard, it is decoration. Three cases cover a non-secret optional, a secret optional, and a plain `Secret`, all run through Alloy's real evaluator so it exercises the same path the config does.

## The takeaway

`fmt.Sprintf("%v", x)` is a fine last resort and a bad default. The moment `x` can be a struct that wraps the thing you actually wanted, `%v` will happily hand you the wrapper. When a value passes through a type system that boxes things — secrets, optionals, capsules — the conversion back out is a real step, not something to leave to default formatting. The change is in [grafana/alloy#6605](https://github.com/grafana/alloy/pull/6605).

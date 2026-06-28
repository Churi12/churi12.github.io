---
layout: post
title: "A CIDR and a list of /32s should be the same thing, but one broke the sidecar"
date: 2026-06-28
author: Miguel Santos
tags: [istio]
---

Someone reported a ServiceEntry that broke their sidecar in a way that made no sense at first. Write the addresses one way and the pod starts. Write the same addresses a different way and the pod never goes ready. The two forms cover the exact same eight IPs:

```yaml
# works: eight explicit hosts
addresses:
- 10.80.68.0/32
- 10.80.68.1/32
# ... through .7/32

# fails: one CIDR for the same eight addresses
addresses:
- 10.80.68.0/29
```

With the `/29`, the sidecar logs a listener it cannot accept:

```
error adding listener: 'virtualOutbound' has duplicate address '0.0.0.0:15001' as existing listener
Listener rejected
Envoy proxy is NOT ready: lds updates: 0 successful, 1 rejected
```

One rejected listener fails the whole LDS update, so the proxy never becomes ready and the pod is stuck. The clue is in the address: `0.0.0.0:15001`. Port 15001 is istio's own outbound capture port, the `virtualOutbound` listener. Somehow this ServiceEntry was generating a second listener on top of it.

## Why the two forms diverge

When istio builds an outbound listener for a service, it has to decide what address to bind. If the service has a concrete IP, it binds that IP. If the address is a CIDR, you cannot bind a range, so it falls back to the wildcard `0.0.0.0` and adds a filter chain match for the CIDR instead:

```go
if !strings.Contains(svcListenAddress, "/") {
    listenerOpts.bind.binds = append([]string{svcListenAddress}, svcExtraListenAddresses...)
} else {
    // Address is a CIDR. Fall back to 0.0.0.0 and filter chain match
    listenerOpts.bind.binds = actualWildcards
    listenerOpts.cidr = ...
}
```

That is the whole divergence. Eight `/32` entries are concrete addresses, so each binds `10.80.68.x:15001`, none of which collide with anything. One `/29` is a CIDR, so it binds `0.0.0.0:15001`, which is precisely where `virtualOutbound` already lives. Same eight IPs, completely different listener address, and only one of them steps on a reserved port.

## There is a guard, and the CIDR walked around it

Istio is not naive about service ports clashing with its own. There is a check, `conflictWithReservedListener`, that knows about 15001 and the other reserved ports, and the listener-building loop already skips services that conflict. So why did this one slip through? Look at how the check bails out early:

```go
if bind != "" {
    if bind != wildcard {
        return false
    }
} else if !protocol.IsHTTP() {
    // bind unspecified and not HTTP: assume it will bind its own address
    return false
}
```

The check runs before the CIDR fallback has happened. At that point the bind is still empty and the protocol is TLS, not HTTP, so it takes the early `return false`: no conflict, this service will bind its own address. That assumption is true for a normal service. It is false for a CIDR service, which is about to be rewritten to the wildcard a few lines later. The guard made its decision based on a bind address that the code then changed out from under it.

## Fix it where the decision actually becomes true

The conflict only becomes real at the moment the listener falls back to the wildcard, so that is where I put the re-check. Right after the CIDR branch sets the wildcard bind:

```go
listenerOpts.bind.binds = actualWildcards
listenerOpts.cidr = append([]string{svcListenAddress}, svcExtraListenAddresses...)
// Now that we are binding to the wildcard address, this listener can
// collide with a reserved listener ... re-check here and skip the port
// if it conflicts.
wildcard := actualWildcards[0]
if canbind, knownlistener := lb.node.CanBindToPort(...); !canbind {
    ...
    return
}
```

Now a CIDR ServiceEntry on a reserved port is skipped, exactly like a concrete-address service on that port already was. It reuses the same `CanBindToPort` the rest of the loop uses, so the behaviour matches rather than inventing a second notion of what conflicts.

## Proving it before trusting it

Before writing the fix I wrote the reproduction, because I wanted to see the bad listener with my own eyes, not just trust the report. The test harness has a `ValidateListeners` helper that already flags duplicate addresses, so a throwaway test that built a CIDR ServiceEntry on port 15001 was enough: it produced `0.0.0.0_15001` and failed, the bug, in a unit test, in under two seconds.

Then I turned that into a proper test with three cases: a CIDR service on 15001 and on 15006 must produce no listener, and a CIDR service on an ordinary port must still get its wildcard listener. That last case matters as much as the first two, because the easy wrong fix is to drop CIDR wildcard listeners too eagerly and quietly break the normal case. I reverted the fix and watched the two reserved-port cases fail while the ordinary-port case kept passing, then put it back and all three passed.

(One small note to my future self: I first wrote that throwaway test with a shell heredoc, and the colorized shell snuck an escape byte into the `.go` file, which the compiler rejected as an illegal character. Write Go from an editor, not a heredoc.)

## The takeaway

When a value gets rewritten partway through a function, any check that ran before the rewrite was answering a question about a state that no longer exists. The conflict guard here was correct for the bind it saw and wrong for the bind the code produced. The fix was not to make the early check smarter about the future, but to ask the question again at the point where the answer actually settles. And the reproduction-first habit paid for itself twice: it confirmed the bug was real before I touched anything, and it caught the over-eager version of the fix that would have broken ordinary CIDR services. The change is in [istio/istio#60722](https://github.com/istio/istio/pull/60722).

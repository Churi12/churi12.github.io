---
layout: post
title: "Istio built the validation context, then refused to authorize it"
date: 2026-06-28
author: Miguel Santos
tags: [istio]
pr_status: merged
---

A Gateway API listener using `OPTIONAL_MUTUAL` TLS termination was resetting every connection. The same listener configured as `MUTUAL`, with the identical secret in the identical namespace, worked. The only difference was one enum value, and that was enough to make istiod log this:

```
warn ads proxy <pod>.<ns> attempted to access unauthorized certificates <cred>-cacert: cross namespace secret reference requires ReferenceGrant
```

The cert secret holds `tls.crt`, `tls.key`, and `ca.crt`. Istio splits that into two SDS resources: the cert itself, and a `-cacert` resource for the CA bundle that validates client certificates. The handshake needs both. `istioctl pc secret` showed the `-cacert` resource stuck in `WARMING`, never delivered, because SDS said the proxy was not allowed to have it.

## Two places decide, and they disagreed

The thing that makes this bug tidy is that the listener is built from one set of rules and authorized by another, and the two had drifted apart.

When istiod builds the TLS context for the listener, it wires the `-cacert` validation context for both mutual modes. That code is explicit about it:

```go
// If tls mode is MUTUAL/OPTIONAL_MUTUAL, create SDS config for gateway/sidecar to fetch certificate validation context
if tlsOpts.Mode == networking.ServerTLSSettings_MUTUAL || tlsOpts.Mode == networking.ServerTLSSettings_OPTIONAL_MUTUAL {
```

So the listener that gets pushed to Envoy references `<cred>-cacert`. Envoy asks SDS for it. SDS checks the request against the set of credentials the proxy is allowed to see, `VerifiedCertificateReferences`, and that set is built somewhere else entirely, in `mergeGateways`:

```go
verifiedCertificateReferences.Insert(rn)
if s.GetTls().GetMode() == networking.ServerTLSSettings_MUTUAL {
    verifiedCertificateReferences.Insert(rn + credentials.SdsCaSuffix)
}
```

The plain cert reference is always added. The `-cacert` suffix is added only for `MUTUAL`. For `OPTIONAL_MUTUAL` the listener asks for a credential that the authorization step never put on the allow-list. One half of the code believed in `OPTIONAL_MUTUAL`, the other half had never heard of it.

That also explains why a `ReferenceGrant` did not help. There are two branches in `mergeGateways`, same-namespace and explicitly-allowed-by-policy, and both gated the `-cacert` insert on `MUTUAL`. No grant could open a door that neither branch was willing to add.

## The fix is to make both sides agree

The authorization step should authorize the same modes the listener-build step builds for. So both inserts now match the expression already used when constructing the context:

```go
if s.GetTls().GetMode() == networking.ServerTLSSettings_MUTUAL ||
    s.GetTls().GetMode() == networking.ServerTLSSettings_OPTIONAL_MUTUAL {
    verifiedCertificateReferences.Insert(rn + credentials.SdsCaSuffix)
}
```

This is not a new idea in the codebase, which is what gave me confidence it would be accepted. The exact same bug existed for `MUTUAL` once, where the validation context was built but the reference was not authorized, and it was fixed the same way. `OPTIONAL_MUTUAL` is just the mode that got left behind when that earlier fix went in. The change is the analog of the one already merged, applied to the mode nobody circled back to.

## Proving it before trusting it

The merge logic has a table-driven test that counts how many references end up authorized. The existing `mutual-cred-in-allowed-ns` case expects two: the cert and its `-cacert`. I added an `optional-mutual-cred-in-allowed-ns` case that is identical except for the mode, and it expects two as well.

Then the step I never skip: I reverted the one-line fix and ran the test. The new case failed with `Expected: 2 Got: 1`, which is the bug stated in numbers, the cert authorized and the CA bundle not. Put the fix back and it reads two. A test you have not watched fail is decoration, not a guard.

## The takeaway

When one part of a system decides what to build and a different part decides what to permit, every new case has to be taught to both, and a list-of-known-modes is exactly the kind of place where one side gets updated and the other does not. The smell here was specific: the same enum, `OPTIONAL_MUTUAL`, named in the context-building code and absent two functions away in the authorization code. Grepping for the places a mode is mentioned, and asking why one site has it and another does not, is often faster than reading the data flow end to end. The change is in [istio/istio#60720](https://github.com/istio/istio/pull/60720), now merged.

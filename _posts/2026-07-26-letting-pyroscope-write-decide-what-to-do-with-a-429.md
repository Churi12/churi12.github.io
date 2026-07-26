---
layout: post
title: "Letting pyroscope.write decide what to do with a 429"
date: 2026-07-26
author: Miguel Santos
tags: [alloy]
---

An Alloy issue asked for something small and reasonable: a way to tell `pyroscope.write` not to retry when the backend returns HTTP 429. By default it retries rate-limited profiles, which is usually right, but if you would rather drop them than pile more requests onto a backend that is already asking you to slow down, there was no way to say so. `loki.write` and `prometheus.remote_write` both already expose a `retry_on_http_429` flag. `pyroscope.write` did not. This is the kind of change I enjoy: the design decision is already made, you just have to match it.

## The behavior was hardcoded

The retry decision lives in `shouldRetry`, and 429 was baked in alongside 408:

```go
func shouldRetry(err error) bool {
    if errors.Is(err, context.DeadlineExceeded) {
        return true
    }

    var writeErr *PyroscopeWriteError
    if errors.As(err, &writeErr) {
        status := writeErr.StatusCode
        if status == http.StatusTooManyRequests || status == http.StatusRequestTimeout {
            return true
        }
        return status >= http.StatusInternalServerError
    }
    // ...
}
```

There is nothing wrong with the logic. It just was not a choice you could make from config.

## Following the precedent instead of inventing one

Because two sibling components already solved this, the interesting part is not the mechanism, it is matching their shape so the option feels native to Alloy. I added the field with the same name, same type, and the same default those components use:

```go
RetryOnHTTP429 bool `alloy:"retry_on_http_429,attr,optional"`
```

Default `true`, set in `GetDefaultEndpointOptions`, so every existing config keeps behaving exactly as it does today. Then `shouldRetry` takes the flag and consults it only for 429:

```go
func shouldRetry(err error, retryOnHTTP429 bool) bool {
    // ...
    if status == http.StatusTooManyRequests {
        return retryOnHTTP429
    }
    if status == http.StatusRequestTimeout {
        return true
    }
    return status >= http.StatusInternalServerError
}
```

Splitting the combined `429 || 408` check into two is the whole change in spirit: 429 becomes configurable, while 408 and 5xx stay unconditionally retried, because those are not what the user asked to control. Both call sites, the push path and the ingest path, pass `ec.options.RetryOnHTTP429` through.

## Proving it before trusting it

I tested this at two levels. `shouldRetry` is pure, so I tested it directly: a 429 is retried with the flag on and dropped with it off, while 408, 503, and 400 return the same answer regardless of the flag. That last part matters as much as the feature itself, since it proves the new knob only touches 429 and leaves the other status codes alone.

Then, because a unit test of a predicate does not prove the flag is actually wired into the retry loop, I added two suite tests that stand up a server always returning 429 and count the requests. With `retry_on_http_429` disabled the profile is sent exactly once. With it enabled the request is retried up to `MaxBackoffRetries` times. Asserting on the observed request count is what convinces me the flag reaches the loop and not just the predicate.

## The takeaway

When a project already has an established convention for a thing, the best contribution is often the least original one: find the precedent, match its name, its default, and its behavior, and make the new option indistinguishable from the ones that came before. A user who already knows `retry_on_http_429` from `loki.write` should not have to learn anything new to use it on `pyroscope.write`. Consistency across siblings is a feature. The change is in [grafana/alloy#6763](https://github.com/grafana/alloy/pull/6763).

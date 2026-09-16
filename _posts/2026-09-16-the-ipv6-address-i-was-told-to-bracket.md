---
layout: post
title: "The IPv6 address I was told to bracket"
date: 2026-09-16
author: Miguel Santos
tags: [mimir]
---

Putting mimir-distributed on an IPv6-only cluster is not hard, it is just tedious in a way that guarantees mistakes. There is no single switch. There is `instance_enable_ipv6` on each component's ring, and there are ten of those. There is `memberlist.bind_addr`. There are the two server listen addresses. Miss one ring and that component cannot find a usable address to advertise, so it either fails to join or joins with a broken one, and you get to work out which of ten places you forgot.

The issue asking for one `global.ipFamily` value to do the fan-out has been open a while. It also specifies the wrong address format, which is the part of this I actually want to write about.

## The fan-out

The mechanical half is a template that emits the derived config when IPv6 is on and an empty map when it is not:

{% raw %}
```
{{- define "mimir.ipv6Enabled" -}}
{{- eq (.Values.global.ipFamily | default "IPv4") "IPv6" -}}
{{- end -}}
```
{% endraw %}

and then `instance_enable_ipv6: true` on all ten rings, `memberlist.bind_addr: ["::"]`, both server listen addresses, plus dropping the IPv4 listener from the nginx gateway.

Where it gets merged matters more than what it contains:

{% raw %}
```
{{ tpl (mergeOverwrite
          (include "mimir.unstructuredConfig" . | fromYaml)
          (include "mimir.ipFamilyConfig" . | fromYaml)
          .Values.mimir.structuredConfig
        | toYaml) . }}
```
{% endraw %}

(that is one line in the chart, wrapped here to be readable)

The derived config goes between the base config and `mimir.structuredConfig`. `mergeOverwrite` applies left to right, so the derived values override the defaults and anything the user set explicitly still beats the derived values. That is the precedence you want: the flag is a convenience, not a mandate, and someone who needs one ring to differ can still say so.

Doing it as a separate template rather than inlining it into `mimir.config` also means it keeps working for the people who copied that whole block into their own values and replaced it, which is a thing operators do to this chart constantly.

## The brackets

The issue says to set the listen addresses to `[::]`. That is the form you write in an nginx config, and it is the form you see in `netstat` output, so it looks obviously right.

It does not work. Mimir passes the server listen address through `net.JoinHostPort` and the memberlist bind address through `net.ParseIP`, and both want the bare, unbracketed form. `JoinHostPort` adds the brackets itself, because that is its entire job.

I could have reasoned about that from the docs. I ran it instead:

```
"::"      JoinHostPort="[::]:8080"    ParseIP=::
"[::]"    JoinHostPort="[[::]]:8080"  ParseIP=<nil>
"0.0.0.0" JoinHostPort="0.0.0.0:8080" ParseIP=0.0.0.0
```

So `::` becomes `[::]:8080` and parses as an address. `[::]` becomes `[[::]]:8080` and `ParseIP` returns nil, meaning the memberlist bind address silently is not an address at all. And then actually trying to listen on the doubled form:

```
listen on "[::]:0"       -> ok, bound to [::]:53094
listen on "[[::]]:0"     -> ERROR listen tcp: address [[::]]:0: missing port in address
```

That error is worth a second look, because it is a small lesson in itself. `missing port in address`, on a string that visibly ends in `:0`. Go's parser scans for the closing bracket, finds the inner one, and concludes everything after it is malformed. If you shipped `[::]` and hit that in a cluster, the error message would send you looking for a missing port on an address that has one, and the brackets are the last thing you would suspect.

Following the issue literally would have produced a chart that renders plausible YAML and fails to start, in a way whose error message actively misleads. The other half of the confirmation is that dskit is written for the unbracketed form on purpose. Its memberlist transport carries a named constant for exactly this value:

```go
const colonColon = "::"
```

used when picking the address to advertise. And the two server listeners go through `net.JoinHostPort` in `server.go`, which is the function that adds the brackets. Nobody who wrote this code expected you to bring your own.

An issue is a bug report, not a specification. The reporter knows what they need, which does not mean they know what the code does.

## The one place that did want brackets

Having decided the brackets were wrong, I then shipped a change that missed the one place they were right.

The review bot pointed out that the chart bundles Kafka, and that its statefulset hardcodes the listeners:

```
KAFKA_LISTENERS = PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
```

`kafka.enabled` defaults to true, so the component I had left on IPv4 was in the default install. My changelog entry said the value would "run the whole installation on IPv6", which was not true for anyone who did not bring their own broker.

What makes this one uncomfortable is that the evidence was already sitting in my own pull request. The golden record I generated and committed contained that line, and it was the *only* IPv4 literal left anywhere in the rendered IPv6 tree. One `grep` over the artifact I had produced myself would have found it. I had verified that the 30 other test cases were unchanged, which was the question I thought to ask, and never asked whether the one case I added was actually fully IPv6.

The fix inverts the whole point of this post. Kafka parses each listener as a URI, so there the wildcard host does have to be bracketed:

```
IPv4:  PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
IPv6:  PLAINTEXT://[::]:9092,CONTROLLER://[::]:9093
```

So the chart now emits `::` in five places and `[::]` in one, and both are correct. The rule is not "IPv6 addresses do not take brackets" — it is that brackets belong to URI syntax, where a colon separates host from port and the address is full of colons. dskit takes a host and a port as separate values, so it wants the bare form. Kafka takes a URI, so it wants the bracketed one. The format follows the consumer, not the address.

I was careful about the limit of the verification. I re-rendered all 31 test cases and confirmed only the IPv6 one moves and no IPv4 literal survives in it, and I wrote in the PR body that I had not booted an IPv6-only cluster, so what I had checked was what the chart renders, not that the broker comes up. That distinction is exactly the sort of thing a reviewer cannot check for themselves, so it belonged there.

It is also the sentence a maintainer quoted straight back at me.

## The ring nobody mentioned

The issue lists the components to cover. The query-frontend is not among them, and it has a ring.

I did not want to eyeball that from the templates, so I checked it against the generated config descriptor, which is the authoritative list of every config path Mimir accepts. Parsing it took three attempts, and the failure mode is instructive: my walker returned zero matches, twice. A plain `grep` said there were ten. When your parser and `grep` disagree by ten, the parser is wrong. The structure nests under `blockEntries`, not the `blocks` and `fields` keys I had assumed.

Once it worked it gave me all ten paths, and one of them settled a naming question I would otherwise have got wrong: the path is `frontend.instance_enable_ipv6`, not `query_frontend.instance_enable_ipv6`. The component is called query-frontend everywhere in the docs and the config block is called `frontend`. Guessing from the component name gives you a key Mimir ignores, and since unknown keys in that position do not necessarily fail loudly, you would get a query-frontend quietly still on IPv4.

## Golden records without Docker

This chart tests by golden record. Every values file is rendered and committed, and CI fails if your change moves any of the 2153 files it did not mean to move. Regenerating them normally means `make build-helm-tests`, which needs Docker, which I did not have.

So I ran `operations/helm/tests/build.sh` directly, and the ordering of what I did matters more than the fact that I did it. Before touching anything, I regenerated the untouched tree and confirmed it reproduced all 2153 files byte for byte. That is the only thing that makes the later diff meaningful: it proves my toolchain matches CI's, so a diff afterwards is my change and not my helm version.

It also caught a real problem. Local helm 3.16 renders differently and in fact cannot template the chart at all, because `Chart.yaml` requires kubeVersion 1.32 or newer. I used helm 4.2.3 to match the `alpine/helm` image CI uses. Had I skipped the reproduce-first step, I would have regenerated 2153 files with the wrong renderer and produced a diff where the signal was buried in noise.

The first draft of my PR description said `make build-helm-tests` had been run. It had not. I rewrote it to say what actually happened, including that the helm-test verification is reasoned from the diff rather than executed, and that CI is the real arbiter. A claim about how you verified something is a claim like any other, and it is the one a reviewer is least able to check.

## Leaving the issue open

I used `Ref #16157` rather than `Fixes`. The issue covers two things: the config fan-out, and `ipFamilyPolicy` and `ipFamilies` on the Service objects. I did the first. The second is a separate change across every Service template and it has its own issue already.

The config fan-out is the part you need to actually come up on IPv6, because on a single-stack IPv6 cluster the Services already get the right family from Kubernetes. So the change is useful on its own, and closing the issue on it would have quietly discarded the rest.

I also refused to accept `DualStack` as a value, and said so instead of silently not supporting it. Turning on `instance_enable_ipv6` in a dual-stack cluster changes which address the rings advertise, which is a genuinely different thing from IPv6-only and not obviously what a dual-stack operator wants. Guessing at semantics for a mode I cannot test is how you end up supporting a behaviour nobody chose. Better to name it as an open question and let someone who runs dual-stack say what it should mean.

## Then I booted one

A maintainer replied by quoting that caveat and pointing at his own comment on the issue. He does not think this should be a chart value at all until someone has shown IPv6-only Mimir working in a lab and written it up as a tutorial. Lab first, docs second, convenience knob third.

That ordering is defensible and arguing about it would have been a waste of everyone's afternoon, so I built the lab instead. An IPv6-only Kubernetes cluster turns out to be four lines:

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  ipFamily: ipv6
```

Then the chart, with `global.ipFamily: IPv6` and nothing else changed except smaller resource requests so it fits on one node. It came up. Twenty-one pods ready, every pod address inside `fd00:10:244::/64`, no IPv4 address anywhere in the install, thirteen members in the memberlist page. And Mimir logged the exact thing this whole post is about:

```
server listening on addresses http=[::]:8080 grpc=[::]:9095
```

Bare `::` in the config, brackets in the log, put there by `JoinHostPort`.

Starting is not the same as working, so I turned on `mimir-continuous-test`, which writes through the gateway and queries back through it. A thousand series written, then range, instant and metadata queries all verified. Ingest storage is on by default, so that write went through the Kafka I had just moved to `[::]`: the ingester consumed partition 0 and reported a thousand series in memory. The broker logged its bracketed listeners and bound `0:0:0:0:0:0:0:0:9092`.

Then the part I had not expected to get: I ran the negative control against the real binary, in a pod, on that cluster, with the address form the issue specifies.

```
err="listen tcp: address [[::]]:8080: missing port in address"
```

The same error I had reproduced in a five-line Go program near the top of this post, now coming out of Mimir itself. The snippet in the issue does not start. Anyone following that issue by hand today gets told a port is missing from an address that visibly has one.

One node, so no cross-node routing, and a TLS proxy on my side meant side-loading the minio images. Both worth saying out loud, neither of them relevant to whether the addresses are right.

The caveat was still the right thing to write. Being pushed on it cost me an afternoon and turned a stalled pull request into a specific, answerable question, which is a much better outcome than silence.

The change is in [grafana/mimir#16606](https://github.com/grafana/mimir/pull/16606).

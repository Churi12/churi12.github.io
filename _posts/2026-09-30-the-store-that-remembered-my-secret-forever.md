---
layout: post
title: "The store that remembered my Secret forever"
date: 2026-09-30
author: Miguel Santos
tags: [alloy]
---

`mimir.alerts.kubernetes` is the Alloy component that watches `AlertmanagerConfig` resources in a cluster and pushes the merged Alertmanager configuration to Mimir. A user reported that it ignored changes to Secrets. They had an `AlertmanagerConfig` with an MS Teams receiver that read its webhook URL from a Secret. They added a key to that Secret, and the component failed every sync with this, until they restarted Alloy:

```
failed to generate Alertmanager configuration: AlertmanagerConfig monitoring/infra:
MSTeamsConfigV2[0]: key "secretkey" in secret "msteams-webhook" not found
```

The key was there. `kubectl get secret` showed it. After a restart it worked straight away. When something is fixed by a restart and nothing else, it is almost always a cache that was built once and never let go.

## Where the Secret gets read

Alloy does not resolve Secret references itself. It uses the config builder from prometheus-operator, the same code the operator uses to render Alertmanager configs, and that builder reads Secrets through an `assets.StoreBuilder`. This is the method that matters:

```go
func (s *StoreBuilder) GetSecretKey(ctx context.Context, namespace string, sel corev1.SecretKeySelector) (string, error) {
	...
	obj, exists, err := s.objStore.Get(sec)
	...
	if !exists {
		secret, err := s.sClient.Secrets(namespace).Get(ctx, sel.Name, metav1.GetOptions{})
		...
		if err = s.objStore.Add(secret); err != nil {
			...
		}
		obj = secret
	}

	secret := obj.(*corev1.Secret)
	if _, found := secret.Data[sel.Key]; !found {
		return "", fmt.Errorf("key %q in secret %q not found", sel.Key, sel.Name)
	}
	...
}
```

It checks its own cache first and only calls the API when the Secret is not cached yet. It has no expiry and no refresh. Once a Secret is read, that copy is the answer for as long as the store exists.

This is not a bug in the store. The doc comment says how it is meant to be used:

```go
// StoreBuilder is a store that fetches and caches TLS materials, bearer tokens
// and auth credentials from configmaps and secrets.
//
// Data can be referenced directly from a Prometheus object or indirectly (for
// instance via ServiceMonitor). In practice a new store is created and used by
// each reconciliation loop.
```

The Alertmanager operator does exactly that. Its `sync` builds a new store at the top of every reconcile. So the cache only lives for one pass: a Secret referenced by five receivers is fetched once instead of five times, and the next reconcile starts empty.

## Where Alloy kept it

In Alloy, `Startup` built one store and gave it to the event processor:

```go
sb := assets.NewStoreBuilder(c.k8sClient.CoreV1(), c.k8sClient.CoreV1())

c.eventProcessor = c.newEventProcessor(queue, informerStopChan, namespaceLister, cfgLister, *baseCfg, sb)
```

and every reconcile reused it. A store meant to live for one reconcile was living for the whole process. The first reconcile cached the Secret as it was then, with no `secretkey` in it, and every later reconcile got that same copy back, and the same "not found". Changing a value that was already there was worse, because nothing failed: Mimir just kept the old webhook URL.

## How it got that way

The shared store was itself a bug fix. Before it, the component passed `nil` as the store, and the first `AlertmanagerConfig` with a Slack receiver crashed Alloy with a nil pointer dereference inside `GetSecretKey`. The fix put a real store in, and put it in the obvious place, next to the other long-lived things the event processor holds. That fixed the crash and nobody noticed the lifetime, because no test ever changed a Secret between two reconciles. A test that never changes a Secret cannot tell a store that lives for one reconcile from one that lives forever.

## The fix

Create the store where it is used, once per reconcile, from the Kubernetes client the event processor already has:

```go
// Use a new store for every reconcile. The store caches every Secret and ConfigMap it reads
// and never reads them again, so a shared one would never see changes to them.
store := assets.NewStoreBuilder(e.kclient.CoreV1(), e.kclient.CoreV1())

cfg, err := e.provisionAlertmanagerConfiguration(ctx, amConfigs, store)
```

The field and the constructor parameter go away. The store is never nil, so the old crash cannot come back. And no Secret informer is needed: the component already reconciles on every `sync_interval`, so a changed Secret is picked up on the next tick.

## The test that fails first

The test does what the user did. It uses a fake clientset with a Secret that has no `api-url` key, and an `AlertmanagerConfig` whose Slack receiver references that key. Then it reconciles three times:

1. With the key missing. This must fail with "not found", and it does, on main and with the fix.
2. After adding the key. On main this still fails with the same "not found", which is the bug. With the fix it succeeds and the URL is in the config sent to Mimir.
3. After changing the value. The new URL must be in the config and the old one must be gone.

I ran it against main before writing the fix, and it failed at step two with the same error as the issue. That is the point of running it first. A test that has only ever passed does not prove anything.

## What I took from it

A cache comes with a lifetime, and you do not see the lifetime at the call site. `NewStoreBuilder` returns something that looks like a client. You can hold it in a struct field, pass it around, and everything works as long as nothing changes. The only place that says "one per reconcile" is a doc comment on the type. When a fix adds a dependency to stop a crash, it is worth asking how long that dependency is meant to live, not just whether it is non-nil.

The change is in [grafana/alloy#7273](https://github.com/grafana/alloy/pull/7273).

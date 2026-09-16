---
layout: post
title: "The autocomplete answer that arrived too late"
date: 2026-09-16
author: Miguel Santos
tags: [tempo]
---

Type a tag value in the TraceQL editor, keep typing, and the suggestion list can end up showing values for something you already finished typing. The list is not wrong, exactly. It is just late. Every keystroke in a value position fires a `tag-values` lookup at Tempo, and nothing anywhere decides which of the answers still matters by the time they come back.

This is the shape of bug I like, because there is no incorrect line of code to point at. Each request is fine. Each response is fine. The defect only exists in the ordering, which means it only shows up on a slow query or a slow network, which means it barely reproduces on a developer laptop and reproduces constantly on a real Tempo with a lot of cardinality.

## Two things go wrong, not one

The first is that a slow response overwrites a fast one. If keystroke three takes 900 ms and keystroke four takes 100 ms, four resolves first and three lands afterwards and wins, because winning here just means being the last promise to resolve. Monaco takes whatever the provider eventually hands it.

The second is that the work is wasted even when the ordering happens to be fine. Monaco knows perfectly well that it no longer wants the completion for keystroke three, and it says so, through a `CancellationToken` it passes to `provideCompletionItems`. The Tempo provider was not accepting that argument at all:

```ts
provideCompletionItems(
  model: monacoTypes.editor.ITextModel,
  position: monacoTypes.Position
): monacoTypes.languages.ProviderResult<monacoTypes.languages.CompletionList> {
```

The signature stops at `position`, so the token was being handed over on every call and dropped on the floor. Monaco was telling us and we were not listening.

## Listening to Monaco

Taking the token is the easy half. `CompletionContext` sits between `position` and the token in the interface, so it has to be named even though nothing uses it:

```ts
provideCompletionItems(
  model: monacoTypes.editor.ITextModel,
  position: monacoTypes.Position,
  _context?: monacoTypes.languages.CompletionContext,
  token?: monacoTypes.CancellationToken
): monacoTypes.languages.ProviderResult<monacoTypes.languages.CompletionList> {
```

Then it gets checked twice, because there are two distinct moments where the answer becomes worthless. Once before doing anything, since Monaco can cancel a request before the provider ever runs:

```ts
// Monaco cancelled this request before we started, so there is no point in asking Tempo anything
if (token?.isCancellationRequested) {
  return { suggestions: [] };
}
```

And once after the network call resolves, which is the check that actually fixes the visible bug:

```ts
return completionItems.then((items) => {
  // Monaco already asked for a newer completion, so this result is stale and we drop it
  if (token?.isCancellationRequested) {
    return { suggestions: [] };
  }
```

That second one is the whole race. The late response still arrives, we just stop pretending it is news.

## Cancelling the request, not only the result

Dropping a stale result is correct but stingy. The HTTP request already went out, Tempo already did the work, the bytes already came back. Grafana gives you a way to not do that, and it is barely documented: put a `requestId` on a `BackendSrvRequest` and a new request with the same id cancels the one in flight. Cancel and replace, for free, as long as the id is stable across the calls that supersede each other and distinct across the calls that do not.

So the id has to be built carefully, and getting it wrong in either direction is a real bug. Too specific and nothing ever cancels anything. Too broad and you cancel requests that were still wanted:

```ts
requestId: `${this.languageProvider.datasource.uid}-${this.instanceId}-traceql-tag-values-${tagName}`,
```

It deliberately does not include the query text, which is what makes it work. Typing another character changes the query but not the id, so the new request cancels the one it supersedes. It does include the tag name, so starting to fill in a different attribute does not kill the lookup for the one you were on.

Then a cancelled request surfaces as an error, and it must not be shown as one:

```ts
if (isFetchError(error) && error.cancelled) {
  // The request was cancelled because a newer one replaced it, nothing went wrong
  return [];
} else if (isFetchError(error)) {
  setAlertText(error.data.error);
```

Without that branch the fix trades a stale dropdown for an error banner on every third keystroke, which is a considerably worse product than the one I started with.

## The part I got wrong

My first version keyed the id on the datasource uid and the tag name. Copilot's automated review said that would make two editors on one dashboard cancel each other. I assumed it was wrong, because bots reliably produce plausible nonsense about concurrency, and went to disprove it.

It was right. `TraceQLEditor.tsx` builds the provider inside `useAutocomplete`:

```tsx
function useAutocomplete(
  // ...
  const providerRef = useRef<CompletionProvider>(
    new CompletionProvider({
```

A ref per hook call, a hook call per editor. So one provider per editor instance, all sharing a datasource uid, all issuing the same id for the same tag name. Two TraceQL panels on a dashboard querying the same Tempo, both completing on `resource.service.name`, and each keystroke in one kills the in-flight request of the other. I would have shipped a fix whose failure mode was strictly worse than the bug, and only on multi-panel dashboards, which is exactly the configuration you do not have open while writing the patch.

The fix is a counter in module scope:

```ts
// Each editor gets its own CompletionProvider, so we number them to keep their request ids apart
let instanceCount = 0;
```

and an id per instance, read once at construction:

```ts
private readonly instanceId = ++instanceCount;
```

Stable across keystrokes within one editor, which is the property cancellation needs. Unique across editors, which is the property correctness needs. Fifteen lines, and the interesting part is that both properties are load-bearing in opposite directions, so there is no way to reason about the id without knowing exactly how many providers exist.

## What I would tell myself

I got to the right answer through a review comment I initially assumed was noise. The lesson is not that the bot is smart, it is that the claim was cheap to check and I nearly did not bother. One `grep` for where the provider is constructed settled it in under a minute, against a fix I had already convinced myself was finished.

The other thing worth saying out loud: the weak point of this change is the tests, not the code. `getOptionsV2` runs the query through `getTemplateSrv()`, so the new tests need `setTemplateSrv` in a `beforeAll` or they throw before they reach the mock. That is global state in a test file, and I do not love it. The alternative was to mock `getOptionsV2` itself, which would have made a tidier test that proved nothing, because the entire behaviour under test is whether the id reaches the datasource layer. A test that cannot fail for the real reason is not worth its tidiness.

The change is in [grafana/grafana-tempo-datasource#259](https://github.com/grafana/grafana-tempo-datasource/pull/259).

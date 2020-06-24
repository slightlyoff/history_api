# Application History State

## Authors:

- Alex Russell <slightlyoff@google.com>
- Tom Wilkinson <twilkinson@google.com>

## Participate
- [Issue tracker](https://github.com/slightlyoff/history_api/issues)
- [Discussion forum](https://github.com/slightlyoff/history_api/issues)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Navigation Events](#navigation-events)
- [UI State Fragments](#ui-state-fragments)
- [Same-Origin History Stack Introspection](#same-origin-history-stack-introspection)
- [Considered alternatives](#considered-alternatives)
  - [A Global History Manager](#a-global-history-manager)
  - [Service Worker Navigation Events](#service-worker-navigation-events)
  - [Built-in Client-Side Routing](#built-in-client-side-routing)
- [Open Design Questions](#open-design-questions)
- [References & acknowledgments](#references--acknowledgments)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Application Developers face a range of challenges when performing client-side updates to dynamic web apps including (but not limited to):

 - Lack of clarity on when to rely on base URL, query parameters, or hash parameters to represent "persistent" state
 - Difficulty in serializing & sharing "UI state" separate from "application state"
 - Complexity in capturing browser-initiated page unloading for [client-side routers](https://blog.risingstack.com/writing-a-javascript-framework-client-side-routing/)
 - Tricky coordination problems between multiple page components which may each want to persist transient UI state but remain largely unaware of each other
 - Difficulty in understanding one's location in, and predicting effects of chagnes to, the [HTML5 History API stack](https://developer.mozilla.org/en-US/docs/Web/API/History_API) due to potentially co-mingled origins

Taken together, these challenges create a "totalizing" effect in applications when client-side state management is introduced. Because a single router system must be responsible for so many aspects, and coordinate so many low-level details, it's challenging to create compatible solutions, or constrain code footprint whilst retaining valuable properties such as lightweight progressive enhancement.

Existing building blocks create subtle, but large, problems.

The History [`pushState`](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState) and [`replaceState`](https://developer.mozilla.org/en-US/docs/Web/API/History/replaceState) APIs provide a mechanism for passing a [cloneable](https://html.spec.whatwg.org/multipage/structured-data.html#structuredserializeinternal) JavaScript state object, which are returned to a page on change in state (usually via user action), however it is left as an exercise to the developer to map potentially different levels of application semantics into this API. It usually "feels wrong" to encode the state of an accordion component's open/close state in _either_ the [path](https://url.spec.whatwg.org/#dom-url-pathname) or [query parameters](https://url.spec.whatwg.org/#dom-url-searchparams), both of which are passed to servers and may have semantic meaning beyond UI state.

URL [hashes](https://url.spec.whatwg.org/#dom-url-hash) are a reasonable next place to look for a solution as they are shareable and not meaningful to server applications by default, but [HTML already assigns meaning](https://html.spec.whatwg.org/multipage/browsing-the-web.html#scroll-to-fragid) to [hashes for anchor elements](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a#Examples), creating potential conflicts. Fragments have also become a [potential XSS vector](https://medium.com/iocscan/dom-based-cross-site-scripting-dom-xss-3396453364fd), in part, because no safe parsing is provided by default. The [`hashchange`](https://developer.mozilla.org/en-US/docs/Web/API/Window/hashchange_event) event can allow components to be notified of state changes, but doesn't provide any semantic for location in stack history or meaningfully integrate into the [History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API).

This leaves application authors torn. Shareable UI state either requires out-of-band mechanisms for persistence, or overloading of URLs, unwanted server-side implications, or potential loss of state when folding information into History API state objects. And that is to say nothing of navigating the APIs themselves and their [various warts](https://github.com/whatwg/html/issues/2174).

Lastly, it should be noted that these problems largely arise only when developers have _already_ exerted enough to control to prevent "normal" link navigations via `<a href="...">`. Naturally-constructed applications will want to progressively enhance anchors, but the current system prevents this, forcing developers to be vigilant to add very specific local event handlers -- or forgo anchor elements, potentially harming accessibility and adding complexity.

## Goals

We hope to enable application developers to:

 - Easily express transient, shareable UI state separately from semantic navigation state
 - Remove complexity and brittleness from responding to requests for state change via links _without_ full page reloads
 - Reduce security concerns regarding UI state serialized to URLs
 - Reduce conflicts with existing libraries for incremental adoption

## Non-goals

These proposals do not seek to:

 - Add a full client-side router to the platform (although it should make these systems easier)
 - Solve all known issues with history introspection and `history.state` durability


## Navigation Events

Currently, client-side libraries need to override the browser's default navigation affordances by [globally hooking nominally unrelated events, e.g. `onclick`](https://github.com/vaadin/vaadin-router/blob/master/src/triggers/click.js#L107). This is [brittle](https://github.com/ReactTraining/react-router/blob/master/packages/react-router-dom/modules/Link.js#L43), as programmatic navigation (`window.location = ...`) can be missed, and may require explicit [decoration of elements to be handled within a system](https://angular.io/guide/router#add-the-router-outlet), complicating progressive enhancement.

To reduce this pain, we propose the cancelable `onbeforenavigate` event.

```js
// Fired *before* window.onbeforeunload

// TODO: should this be on `window.location`` instead?
// It's not currently an EventTarget
history.onbeforenavigate = (e) => {
  // Cancel navigation; only available for same-origin navigations
  e.preventDefault();
  console.log(e.url); // the destination URL; TODO: precedent?
};
```

The `onbeforenavigate` event is cancellable for all navigations that would affect _the current document context_. It's an open question as to whether or not it should be cancellable for links with the form `<a href="..." target="_blank">...</a>`.

`onbeforenavigate` also composes with form submission; for example, consider this form:

```html
<form method="POST" action="/app/submit">
  <input type="hidden" name="id" value="whatevs">
  <label for="content">Content:</label>
  <textarea id="content">
    ...
  </textarea>
  <button type="submit">Send!</button>
</form>
```

Submission can be detected (in addition to the non-standard [bubbling behavior](https://source.chromium.org/chromium/chromium/src/+/master:third_party/blink/web_tests/fast/events/onsubmit-bubbling.html)) like this:

```js
history.onbeforenavigate = (e) => {
  // Cancel navigation; only available for same-origin navigations
  e.preventDefault();

  // Check to see if the source of the navigation is form submission:
  if (e.target.tagName == "FORM") {
    let form = e.target;

    // As we're handling it here, make sure other event handlers don't get a
    // crack at the event:
    e.stopImmediatePropagation();

    // Submit the form via Ajax instead:
    fetch(form.action, {
      method: form.method,
      body: new FormData(form)
    }).then(
      // Update UI with response
    );
    // Set loading UI here
  }
};
```

## UI State Fragments

UI State Fragments build on the recently-launched [Scroll-to-Text Fragment](https://github.com/WICG/scroll-to-text-fragment) syntax. For those not familiar, Scroll-to-Text Fragments enable browsers to highlight portions of a page using a syntax like:

```
https://example.com/#:~:text=prefix-,startText,endText,-suffix
```

This syntax was [designed to be extensible](https://github.com/WICG/scroll-to-text-fragment#multiple-text-directives) via the `&` joiner, e.g.:

```
https://example.com/#:~:text=sometext&anotherdirective=value&etc...
```

We build on this to encode a single (page-level) serialised object in a (name open for bikeshedding) `uistate` directive, e.g.:


```
https://example.com/#:~:uistate=<value>
```

The value itself is the result of [URI component encoding](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent) a [JSON serialization](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify) of the subset of Structure Cloneable properties that can be naturally represented in JSON. In code, that makes these lines roughly equivalent:

```js
let state = { "foo": true, "bar": [ 1, "ten" ] };
window.location.hash = `#:~:uistate=${
  encodeURIComponent(JSON.stringify(state))
}`;

```

Is long-hand for:

```js
history.uistate = { "foo": true, "bar": [ 1, "ten" ] };
```

Both result in a URL like:

```
https://example.com/#:~:uistate=%7B"foo"%3Atrue%2C"bar"%3A%5B1%2C"ten"%5D%7D
```

Updates to `uistate` _do not persist to the history stack_.

Pages can use `uistate` to re-construct visual context by consulting the (browser parsed) `history.uistate` property at any point; e.g.:

```js
myAppJs.setInitialUIState(history.uistate);
```

## Same-Origin History Stack Introspection

The history stack has a long list of documented issues, chief among them mixing between first and third party contexts, which renders the [`go()`](https://developer.mozilla.org/en-US/docs/Web/API/History/go) and [`back()`](https://developer.mozilla.org/en-US/docs/Web/API/History/back) methods nearly pointless.

We propose extensions to the history stack to provide visibility into same-origin stack state and extensions to existing methods to iterate through them. The following snippet captures the early proposal:

```js
// Returns an iterator of HistoryStateEntries (a new type)
let initialState = history.current; // wraps `.state`; name TBD
let currentState = history.pushSatate({ foo: 1 }, "", ""); // return value
let sameOriginStack = history.states(); // TODO: async? array?
console.log(history.length);  // perhaps 5
console.log(sameOriginStack.length); // can be less
let item = null;
for (item of sameOriginStack) {
  console.log(currentState === item); // `true` for first iteration
  console.log(item.url);
  console.log(item.state); // the state in history.state
}
history.go(item); // new parameter type overload
```

> Lots of "spelling" issues to be worked out here

## Considered alternatives

### A Global History Manager

We have previously considered (in private docs) a more fullsome rewrite of the history APIs to make it clearer that a single "installable" history manager should own potential navigation decisions. This might turn into the correct alternative as we explore, however this document seeks to find smaller potential changes in the short run.

### Service Worker Navigation Events

It's possible control the responses for a document's content inside a Service Worker, and there are [proposals](https://github.com/WICG/sw-launch) for handling some subset of navigation disposition (new window? new tab? re-use?) from within that context. These aren't particularly satisfyign from both a performance and programmability perspecitve. Service Workers may be shut down and would need to be restarted to handle decisions about if/how to navigate. They also lack data to most DOM data, including form information, making it complex to offload enough state to them to make decisions. It also isn't clear that it's good layering to invoke them here.

As a final reason not to go this route, Service Workers install asynchronously and may not be available in time to catch some navigations. The proposed deisgn does not have this problem.

### Built-in Client-Side Routing

A new type of link could explicitly invoke client side routing, or we could imagine a more maximal route-pre-definition system inside the document to tell the system to "skip" full navigations to specific routes. This somewhat-more-declaraitve approach could give the UA a larger role in helping users decide what they want an app to do at the limit, however we don't have concrete need for this at the moment and such a system could be layered on later without much concern.

## Open Design Questions

  - naming for `history.uistate` and `onbeforenavigate` is very much TBD
  - positional param changes to `pushState` and `replaceState`?
  - we don't _think_ the new history stack introspection API adds more fingerprinting attacks, but need to validate

## References & acknowledgments

Many thanks for valuable feedback and advice from:

- Dima Voytenko
- David Bokan
- Jake Archibald

Prior work in this area includes:

 - Many, many client-side routing packages (TODO: enumerate)
 - the [HTML5 History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API) and associated [`hashchange` event](https://developer.mozilla.org/en-US/docs/Web/API/Window/hashchange_event)

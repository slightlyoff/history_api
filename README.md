# Application History State

## Authors:

- Alex Russell <slightlyoff@google.com>
- Tom Wilkinson <twilkinson@google.com>


## Participate
- [Issue tracker](https://github.com/slightlyoff/history_api/issues)
- [Discussion forum](https://github.com/slightlyoff/history_api/issues)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Application Developers face a range of challenges when performing client-side updates to dynamic web apps including (but not limited to):

 - Lack of clarity on when to rely on base URL, query parameters, or hash parameters to represent "persistent" state
 - Difficulty in serializing & sharing "UI state" separate from "application state"
 - Complexity in capturing browser-initiated page unloading for [client-side routers](https://blog.risingstack.com/writing-a-javascript-framework-client-side-routing/)
 - Tricky coordination problems between multiple page components which may each want to persist transient UI state but remain largely unaware of each other
 - Difficulty in understanding one's location in, and predicting effects of chagnes to, the [HTML5 History API stack](https://developer.mozilla.org/en-US/docs/Web/API/History_API) due to potentially co-mingled origins

Taken together, these challenges create a "totalizing" effect in applications when client-side state management is introduced. Because a single router system must be responsible for so many aspects, and coordiate so many low-level details, it's challenging to create compatible solutions, or constrain code footprint whilst retinaining valuable properties such as lightweight progressive enhancement.

Existing building blocks create subtle, but large, problems.

The History [`pushState`](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState) and [`replaceState`](https://developer.mozilla.org/en-US/docs/Web/API/History/replaceState) APIs provide a mechansim for passing a [cloneable](https://html.spec.whatwg.org/multipage/structured-data.html#structuredserializeinternal) JavaScript state object, which are returned to a page on change in state (usually via user action), however it is left as an exercise to the developer to map potentially different levels of application semantics into this API. It usually "feels wrong" to encode the state of an accordion component's open/close state in _either_ the [path](https://url.spec.whatwg.org/#dom-url-pathname) or [query parameters](https://url.spec.whatwg.org/#dom-url-searchparams), both of which are passed to servers and may have semantic meaning beyond UI state.

URL [hashes](https://url.spec.whatwg.org/#dom-url-hash) are a reasonable next place to look for a solution as they are shareable and not meanignful to server applications by default, but [HTML already assigns meaning](https://html.spec.whatwg.org/multipage/browsing-the-web.html#scroll-to-fragid) to [hashes for anchor elements](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a#Examples), creating potential conflicts. Fragments have also become a [potential XSS vector](https://medium.com/iocscan/dom-based-cross-site-scripting-dom-xss-3396453364fd), in part, because no safe parsing is provided by default. The [`hashchange`](https://developer.mozilla.org/en-US/docs/Web/API/Window/hashchange_event) event can allow components to be notified of state changes, but doesn't provide any semantic for location in stack history or meaningfully integrate into the [History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API).

This leaves application authors torn. Shareable UI state either requires out-of-band mechanisms for persistence, or overloading of URLs, unwanted server-side implications, or potential loss of state when folding information into History API state objects. And that is to say nothing of navigating the APIs themselves and their [various warts](https://github.com/whatwg/html/issues/2174).

Lastly, it should be noted that these problems largely arise only when developers have _already_ exerted enough to control to prevent "normal" link navigations via `<a href="...">`. Naturally-constructed applications will want to progressively enhance anchors, but the current system prevents this, forcing developers to be vigilant to add very specific local event handlers -- or forego anchor elements, potentially harming accessiblity and adding complexity.

## Goals

We hope to enable application developers to:

 - Easily express transient, shareable UI state separately from semantic navigation state
 - Remove complexity and brittleness from responding to requests for state change via links _without_ full page reloads
 - Reduce security concerns regarding UI state serialized to URLs
 - Reduce conflicts with existing libraries for incremental adoption

## Non-goals

These proposals do not seek to:

 - Add a full client-side router to the platform (although it should make these systems easier)


## Navigation Events

Currently, client-side libraries need to override the browser's default navigation affordances by [globally hooking nominally unrelated events, e.g. `onclick`](https://github.com/vaadin/vaadin-router/blob/master/src/triggers/click.js#L107). This is [brittle](https://github.com/ReactTraining/react-router/blob/master/packages/react-router-dom/modules/Link.js#L43), as programmatic navigation (`window.location = ...`) can be missed, and may require explicit [decoration of elements to be handled within a system](https://angular.io/guide/router#add-the-router-outlet), complicating progressive enhancement.

To reduce this pain, we propose the cancelable `onbeforenavigate` event.

```js
// Fired *before* window.onbeforeunload

// TODO: should this be on `window.location`` instead?
window.history.onbeforenavigate = (e) => {
  // Cancel navigation; only available for same-origin navigations
  e.preventDefault();
  // TODO: decide how to map in SubmitEvent and formdata
  console.log(e.url); // the destination URL; TODO: precident?
};
```

## UI State Fragements

UI State Fragments build on the recently-launched Scroll-to-Text Fragment syntax.

```js

```



## Considered alternatives

TODO(slightlyoff): need to discuss:

  - a global history manager router
  - Direct changes to `pushState` and `replaceState`
  -


### [Alternative 1]

[Describe an alternative which was considered,
and why you decided against it.]

### [Alternative 2]

[etc.]

## Open Design Questions

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Dima Voytenko
- David Bokan
- <TODO>

Prior work in this area includes:

 - Many, many client-side routing packages
 - the [HTML5 History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API) and associated [`hashchange` event](https://developer.mozilla.org/en-US/docs/Web/API/Window/hashchange_event)

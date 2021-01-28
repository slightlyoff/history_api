# App History API

The web's existing [history API](https://developer.mozilla.org/en-US/docs/Web/API/History) is problematic for a number of reasons, which makes it hard to use for web applications. This proposal introduces a new one, which is more directly usable by web application developers to address the use cases they have for history introspection, mutation, and observation/interception.

This new API layers on top of the existing API and specification infrastructure, with well-defined interaction points. The main differences are that it is scoped to the current origin and frame, and it is designed to be pleasant to use instead of being a historical accident with many sharp edges.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of contents

- [Problem statement](#problem-statement)
- [Goals](#goals)
- [Proposal](#proposal)
  - [The current entry, and single-page navigations](#the-current-entry-and-single-page-navigations)
  - [Inspection of the app history list](#inspection-of-the-app-history-list)
  - [Navigation through the app history list](#navigation-through-the-app-history-list)
  - [Navigation monitoring and interception](#navigation-monitoring-and-interception)
    - [Measuring standardized standardized single-page navigations](#measuring-standardized-standardized-single-page-navigations)
    - [Example: replacing navigations with single-page app navigations](#example-replacing-navigations-with-single-page-app-navigations)
    - [Example: single-page app "redirects"](#example-single-page-app-redirects)
    - [Example: affiliate links](#example-affiliate-links)
  - [Queued up single-page navigations](#queued-up-single-page-navigations)
  - [Per-entry events](#per-entry-events)
  - [Current entry change monitoring](#current-entry-change-monitoring)
  - [Complete event sequence](#complete-event-sequence)
- [Guide for migrating from the existing history API](#guide-for-migrating-from-the-existing-history-api)
  - [Performing navigations](#performing-navigations)
  - [Using `navigate` handlers plus non-history APIs](#using-navigate-handlers-plus-non-history-apis)
  - [Attaching and using history state](#attaching-and-using-history-state)
  - [Introspecting the history list](#introspecting-the-history-list)
  - [Watching for navigations](#watching-for-navigations)
- [Integration with the existing history API and spec](#integration-with-the-existing-history-api-and-spec)
- [Impact on back button and user agent UI](#impact-on-back-button-and-user-agent-ui)
- [Security and privacy considerations](#security-and-privacy-considerations)
- [Stakeholder feedback](#stakeholder-feedback)
- [Acknowledgments](#acknowledgments)
- [Appendix: full API surface, in Web IDL format](#appendix-full-api-surface-in-web-idl-format)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Problem statement

Web application developers, as well as the developers of router libraries for single-page applications, want to accomplish a number of use cases related to history:

- Intercepting cross-document navigations, replacing them with single-page navigations (i.e. loading content into the appropriate part of the existing document), and then updating the URL bar.

- Performing single-page navigations that create and pushing a new entry onto the history list, to represent a new conceptual history entry.

- Navigating backward or forward through the history list via application-provided UI.

- Synchronizing application or UI state with the current position in the history list, so that user- or application-initiated navigations through the history list appropriately restore application/UI state.

The existing [history API](https://developer.mozilla.org/en-US/docs/Web/API/History) is difficult to use for these purposes. The fundamental problem is that `window.history` surfaces the joint session history of a browsing session, and so gets updated in response to navigations in nested frames, or cross-origin navigations. Although this view is important for the user, especially in terms of how it impacts their back button, it doesn't map well to web application development. A web application cares about its own, same-origin, current-frame history entries, and having to deal with the entire joint session history makes this very painful. Even in a carefully-crafted web app, a single iframe can completely mess up the application's history.

The existing history API also has a number of less-fundamental, but still very painful, problems around how its API shape has grown organically, with only very slight considerations for single-page app architectures. For example, it provides no mechanism for intercepting navigations; to do this, developers have to intercept all `click` events, cancel them, and perform the appropriate `history.pushState()` call. The `history.state` property is a very bad storage mechanism for application and UI state, as it disappears and reappears as you transition throughout the history list, instead of allowing access to earlier entries in the list. And the ability to navigate throughout the list is limited to numeric offsets, with `history.go(-2)` or similar; thus, navigating back to an actual specific state requires keeping a side table mapping history indices to application states.

To hear more detail about these problems, in the words of a web developer, see [@dvoytenko](https://github.com/dvoytenko)'s ["The case for the new Web History API"](https://github.com/dvoytenko/web-history-api/blob/master/problem.md). See also [@housseindjirdeh](https://github.com/housseindjirdeh)'s ["History API and JavaScript frameworks"](https://docs.google.com/document/d/1gLW_FlR_wD93ZWXWmH14q0UssBaR0eGMk8njyr6p3cE/edit).

## Goals

Overall, our guiding principle is to make it easy for web application developers to write applications which give good user experiences in terms of the history list, back button, and other navigation UI (such as open-in-new-tab). We believe this is too hard today with the `window.history` API.

From an API perspective, our primary goals are as follows:

- Allow easy conversion of cross-document navigations into single-page app same-document navigations, without fragile hacks like a global `click` handler.

- Provide a uniform way to signal single-page app navigations, including their duration.

- Provide a reliable system to tie application and UI state to history entries.

- Continue to support the pattern of allowing the history list to contain state that is not serialized to the URL. (This is possible with `history.pushState()` today.)

- Provide events for notifying the application about navigations through the list of history entries, which they can use to synchronize application or UI state.

- Allow analytics (first- or third-party) to watch for navigations, including gathering timing information about how long they took, without interfering with the rest of the application.

- Provide a way for an application to reliably navigate through its own history list.

- Provide a reasonable layering onto and integration with the existing `window.history` API, in terms of spec primitives and ensuring non-terrible behavior when both are used.

Non-goals:

- Allow web applications to intercept navigations which they cannot intercept today (e.g., URL bar or back button navigations).

- Provide applications knowledge of cross-origin history entries or state.

- Provide applications knowledge of other frames' entries or state.

- Provide platform support for the coordination problem of multiple routers (e.g., per-UI-component routers) on a single page. We plan to leave this coordination to frameworks for now (with the frameworks using the new API).

- Handle the case where the Android back button is being used as a "modal close signal"; instead, we believe that's best handled by [a separate API](./history_and_modals.md).

- Provide any handling for preventing navigations that might lose data: this is already handled orthogonally by the platform's `beforeunload` event.

- Provide an elegant layering onto or integration with the existing `window.history` API. That API is quite problematic, and we can't be tied down by a need to make every operation in the new API isomorphic to one in the old API.

A goal that might not be possible, but we'd like to try:

- It would be ideal if this API were polyfillable, especially in its mainline usage scenarios.

Finally, although it's really a goal for all web APIs, we want to call out a strong focus on interoperability, backstopped by [web platform tests](http://web-platform-tests.org/). The existing history API and its interactions with navigation have terrible interoperability (see [this vivid example](https://docs.google.com/document/d/1Pdve-DJ1JCGilj9Yqf5HxRJyBKSel5owgOvUJqTauwU/edit#)). We hope to have solid and well-tested specifications for:

- Every aspect and self-interaction of the new API

- Every aspect of how the new API integrates and interacts with the `window.history` API (including things like relative timing of events)

Additionally, we hope to drive interoperability through tests, spec updates, and browser bugfixes for the existing `window.history` API while we're in the area, to the extent that is possible; some of this work is being done in [whatwg/html#5767](https://github.com/whatwg/html/issues/5767).

## Proposal

### The current entry, and single-page navigations

The entry point for the app history API is `window.appHistory`. Let's start with `appHistory.currentEntry`, which is an instance of the new `AppHistoryEntry` class. This class has the following readonly properties:

- `key`: a user-agent-generated UUID identifying this history entry. In the past, applications have used the URL as such a key, but the URL is not guaranteed to be unique.

- `url`: the URL of this history entry (as a string).

- `state`: returns the application-specific state stored in the history entry (or `null` if there is none).

- `sameDocument`: a boolean indicating whether this entry is for the current document, or whether navigating to it will require a full navigation (either from the network, or from the browser's back/forward cache). Note: for `appHistory.currentEntry`, this will always be `true`.

_NOTE: `state` would benefit greatly from having an interoperable size limit. This would depend on [whatwg/storage#110](https://github.com/whatwg/storage/issues/110)._

For single-page applications that want to update the current entry in the same manner as today's `history.replaceState()` APIs, we have the following:

```js
// Updates the URL shown in the address bar, as well as the url property. `state` stays the same.
appHistory.updateCurrentEntry({ url });

// You can also explicitly null out the state, instead of carrying it over:
appHistory.updateCurrentEntry({ url, state: null });

// Only updates the state property.
appHistory.updateCurrentEntry({ state });

// Update both at once.
appHistory.updateCurrentEntry({ url, state });
```

_TODO: more realistic example, maybe something Redux-esque like `appHistory.updateCurrentEntry({ state: {...appHistory.currentEntry.state, newKey: newValue } })`._

Similarly, to push a new entry in the same manner as today's `history.pushState()`, we have the following:

```js
// Pushes a new entry onto the app history list, copying the URL (but not the state) from the current one.
await appHistory.pushNewEntry();

// If you want to copy over the state, you can do so explicitly:
await appHistory.pushNewEntry({ state: appHistory.currentEntry.state });

// Copy over the URL, and set a new state value:
await appHistory.pushNewEntry({ state });

// Use a new URL, resetting the state to null:
await appHistory.pushNewEntry({ url });

// Use a new URL and state:
await appHistory.pushNewEntry({ url, state });
```

As with `history.pushState()` and `history.replaceState()`, the new URL here must be same-origin and only differ in the path, query, or fragment portions from the current document's current URL. And as with those, you can use relative URLs.

Note that `appHistory.pushNewEntry()` is asynchronous. As with other [navigations through the app history list](#navigation-through-the-app-history-list), pushing a new entry can be [intercepted or canceled](#navigation-monitoring-and-interception), so it will always be delayed at least one microtask.

In general, you would use `appHistory.updateCurrentEntry()` and `appHistory.pushNewEntry()` in similar scenarios to when you would use `history.pushState()` and `history.replaceState()`. However, note that in the app history API, there are some cases where you don't have to use `appHistory.pushNewEntry()`; see [the discussion below](#using-navigate-handlers-plus-non-history-apis) for more on that subject.

Crucially, `appHistory.currentEntry` stays the same regardless of what iframe navigations happen. It only reflects the current entry for the current frame. The complete list of ways the current app history entry can change are:

- Via the above APIs, used by single-page apps to manage their own history entries.

- A fragment navigation, which will act as `appHistory.pushNewEntry({ url: urlWithFragment, state: appHistory.currentEntry.state })`, i.e. it will copy over the state.

- A full-page navigation to a different document. This could be an existing document in the browser's back/forward cache, or a new document. In the latter case, this will generate a new entry on the new page's `window.appHistory` object, somewhat similar to `appHistory.pushNewEntry({ url: navigatedToURL, state: null })`. Note that if the navigation is cross-origin, then we'll end up in a separate app history list for that other origin.

Finally, note that these APIs also have a callback-based variant for dealing with queued navigations. We discuss those [below](#queued-up-single-page-navigations), since examples involving them are easier to write after we have shown the basics of [navigation interception](#navigation-monitoring-and-interception).

### Inspection of the app history list

In addition to the current entry, the entire list of app history entries can be inspected, using `appHistory.entries`, which returns a frozen array of `AppHistoryEntry` instances. (Recall that all app history entries are same-origin contiguous entries for the current frame, so this is not a security issue.)

This solves the problem of allowing applications to reliably store state in an `AppHistoryEntry`'s `state` property: because they can inspect the values stored in previous entries at any time, it can be used as real application state storage, without needing to keep a side table like one has to do when using `history.state`.

In combination with the following section, the `entries` API also allows applications to display a UI allowing navigation through the app history list.

### Navigation through the app history list

The way for an application to navigate through the app history list is using `appHistory.navigateTo(key)`. For example:

_TODO: realistic example of when you'd use this._

Unlike the existing history API's `history.go()` method, which navigates by offset, navigating by key allows the application to not care about intermediate history entries; it just specifies its desired destination entry.

There are also convenience methods, `appHistory.back()` and `appHistory.forward()`.

All of these methods return promises, because navigations can be intercepted and made asynchronous by the `navigate` event handlers that we're about to describe in the next section. There are then several possible outcomes:

- The `navigate` event handler marks the navigation as failed, in which case the promise rejects as they indicate, and `appHistory.currentEntry` stays the same.

- The `navigate` event cancels the navigation, in which case the promise fulfills with `undefined` and `appHistory.currentEntry` stays the same.

- It's not possible to navigate to the given entry, e.g. `appHistory.navigateTo(key)` was given a non-existant `key`, or `appHistory.back()` was called when there's no previous entries in the app history list. In this case also, the promise fulfills with `undefined` and `appHistory.currentEntry` stays the same.

- The navigation succeeds, and was a same-document navigation. Then the promise fulfills with `undefined`,  and `appHistory.currentEntry` (as well as the URL bar, if appropriate) will update.

- The navigation succeeds, and it was a different-document navigation. Then the promise will never settle, because the entire document and all its promises will disappear.

### Navigation monitoring and interception

The most interesting event on `window.appHistory` is the one which allows monitoring and interception of navigations: the `navigate` event. It fires on any navigation, either user-initiated or application-initiated, which would update the value of `appHistory.currentEntry`. This includes cross-origin navigations (which will take us out of the current app history list), but it does _not_ include navigations which target other windows, e.g. right click/open in new tab. **We expect this to be the main event used by application- or framework-level routers.**

The event object has several useful properties:

- `userInitiated`: a boolean indicating whether the navigation is user-initiated (i.e., a click on an `<a>`, or a form submission) or application-initiated (e.g. `location.href = ...`, `appHistory.navigateTo(...)`, etc.).

- `destinationEntry`: an `AppHistoryEntry` containing the information about the destination of the navigation. Note that this entry might or might not yet be in `window.appHistory.entries`; if it is not, then its `state` will be `null`.

- `sameOrigin`: a convenience boolean indicating whether the navigation is same-origin, and thus will stay in the same app history or not. (I.e., this is `(new URL(e.destinationEntry.url)).origin === self.origin`.)

- `fragmentTarget`: a DOM element or null, indicating the target of the current navigation, if the current navigation is a same-document [fragment navigation](https://html.spec.whatwg.org/#scroll-to-fragid) or [scroll to text fragment navigation](https://github.com/WICG/scroll-to-text-fragment). _TODO I don't this works for scroll to text fragment, which uses ranges, not elements. Maybe make this just a boolean._

- `formData`: a [`FormData`](https://developer.mozilla.org/en-US/docs/Web/API/FormData) object containing form submission data, or `null` if the navigation is not a form submission.

Note that you can check if the navigation will be same-document via `event.destinationEntry.sameDocument`.

In some cases, the event is cancelable via `event.preventDefault()`, which prevents the navigation from going through. Specifically:

- It is cancelable for user-initiated navigations via `<a>` elements, including both same-document fragment navigations and cross-document navigations.
- It is cancelable for programmatically-initiated navigations, via mechanisms such as `location.href = ...` or `aElement.click()`, including both same-document fragment navigations and cross-document navigations.
- It is cancelable for programmatically-initiated same-document navigations initiated via `appHistory.pushNewEntry()`, `appHistory.updateCurrentEntry()`, or their old counterparts `history.pushState()` and `history.replaceState()`.
- It is _not_ cancelable for user-initiated navigations via browser UI, such as the URL bar or bookmarks.
- It is _not_ cancelable for programmatically-initiated navigations originating from other windows, such as `window.open(url, "nameOfAnotherWindow")`.

_TODO: should the event even fire at all, for these latter two cases?_

These restrictions are designed so that canceling the `navigate` event gives web developers an easier mechanism to do things they can already do. That is, web developers can already intercept `<a>` `click` events, or modify their code that would set `location.href`. It does not give them any new powers, and in particular it does not allow trapping the user on a page by intercepting the back button or URL bar navigations.

Additionally, the event has a special method `event.respondWith(promise)`. This can only be called when the event is cancelable; in other cases it will throw an exception. If called, this will:

- Cancel the navigation (but see below).
- Wait for the promise to settle.
  - If it rejects, stay on the current app history entry.
  - If it fulfills, push/replace the destination app history entry onto/in the app history list.
    - Note that this means that some parts of the navigation, namely updating the URL and state, do go through, just in a delayed manner.
    - But other parts, like unloading the document and fetching a new one, or scrolling to a fragment, were indeed canceled.
    - Notably, this means that for navigations caused by `appHistory.pushNewEntry()`, `appHistory.updateCurrentEntry()`, `history.pushState()`, or `history.replaceState()`, whose only effect is updating the history entry, the navigation was just delayed, not really canceled.
- For the duration of the promise settling, any browser loading UI such as a spinner will behave as if it were doing a cross-document navigation.
- After the promise settles, the browser will update its UI (such as URL bar or back button) to reflect the new current history entry.

_TODO: should we give direct control over when the browser UI updates, in case developers want to update it earlier in the lifecycle before the promise fully settles? E.g. `event.commitNavigation()` or `event.commitNavigationUI()`? Would it be OK to let the UI get out of sync with the history list?_

#### Measuring standardized standardized single-page navigations

The `navigate` event's `event.respondWith()` method provides a helpful convenience for implementing single-page navigations, as discussed above. But beyond that, providing a direct signal to the browser as to the duration and outcome of a single-page navigation has wider ecosystem benefits in regards to metrics gathering.

In particular, today it is not clear to the browser when a user interaction causes a single-page navigation, because of the app-specific JavaScript that intermediates between such interactions and the eventual call to `history.pushState()`/`history.replaceState()`. It is also unclear exactly when the navigation ends: e.g., some applications optimistically call `history.pushState()` before the content is loaded.

Using `respondWith()` solves these problems. It gives the browser clear insight into when a navigation is being handled as a single-page navigation, and the provided promise allows the browser to know how long the navigation takes, and whether or not it succeeds. We expect browsers to use these to update their own UI (including any loading indicators; see [whatwg/fetch#19](https://github.com/whatwg/fetch/issues/19) and [whatwg/html#330](https://github.com/whatwg/html/issues/330) for previous feature requests).

Additionally, analytics frameworks would be able to consume this information from the browser in a way that works across all applications using the app history API. See the example in the [Current entry change monitoring](#current-entry-change-monitoring) section for one way this could look; other possibilities include integrating into the existing [performance APIs](https://w3c.github.io/performance-timeline/).

This standardized notion of single-page navigations also gives a hook for other useful metrics to build off of. For example, you could imagine variants of the `"first-paint"` and `"first-contentful-paint"` APIs which are collected after the promise provided to `respondWith()` settles. Or, you could imagine vendor-specific or application-specific measurements like [Cumulative Layout Shift](https://web.dev/cls/) or React hydration time being reset after such navigations.

This isn't a complete panacea: in particular, metrics derived in such a way are potentially "gameable" using techniques such as

```js
event.respondWith(Promise.resolve());
nowDoTheActualLoadingWork();
```

(which makes the navigation appear instant to the browser). Another potential way of driving down average measured "load time" would be by generating excessive `navigate` events that don't actually do anything. So in scenarios where the web application is less interested in measuring itself, and more interested in driving down specific metrics, those creating the metrics will need to take into account such misuse of the API. Some potential countermeasures against such gaming could include:

- Filtering to only count navigations where `event.userInitiated` is true.

- Filtering to only count navigations where the URL changes (i.e., `appHistory.currentEntry.url !== event.destinationEntry.url`).

- We hope that most analytics vendors will come to automatically track `navigate` events as page views, and measure their duration. Then, apps using such analytics vendors would have an incentive to keep their page view statistics meaningful, and thus be disincentivized to generate spurious navigations.

- To the extent developers want to get the correct browser UI treatment for a navigation, including the loading spinner, they will need to pass a promise containing accurate timing and success/failure information to `respondWith()`. Gaming metrics by passing fake promises would lose this nice browser affordance.

#### Example: replacing navigations with single-page app navigations

The following is the kind of code you might see in an application or framework's router:

```js
appHistory.addEventListener("navigate", e => {
  // Don't intercept cross-origin navigations; let the browser handle those normally.
  if (!e.sameOrigin) {
    return;
  }

  // Don't intercept scroll-to-fragment or scroll-to-text-fragment navigations.
  if (e.fragmentTarget) {
    return;
  }

  if (e.formData) {
    e.respondWith(processFormDataAndUpdateUI(e.formData));
  } else {
    e.respondWith(doSinglePageAppNav(e.destinationEntry));
  }
});
```

Here, `doSinglePageAppNav` and `processFormDataAndUpdateUI` are functions that can return a promise. For example:

```js
async function doSinglePageAppNav(destinationEntry) {
  const htmlFromTheServer = await (await fetch(destinationEntry.url)).text();
  document.querySelector("main").innerHTML = htmlFromTheServer;
}
```

Note how this example responds to various types of navigations:

- Cross-origin navigations: let the browser handle it as usual
- Same-document fragment navigations: let the browser handle it as usual
- Same-document URL/state updates (via `history.pushState()`, `appHistory.updateCurrentEntry()`, etc.):
  1. Send the information about the URL/state update to `doSinglePageAppNav()`, which will use it to modify the current document
  1. After that UI update is done, potentially asynchronously, update the history list and browser UI
- Cross-document normal navigations:
  1. Prevent the browser handling, which would unload the document and create a new one from the network
  1. Instead, send the information about the navigation to `doSinglePageAppNav()`, which will use it to modify the current document
  1. After that UI update is done, potentially asynchronously, update the history list and browser UI
- Cross-document form submissions:
  1. Prevent the browser handling, which would unload the document and create a new one from the network
  1. Instead, send the form data to `processFormDataAndUpdateUI()`, which will use it to modify the current document
  1. After that UI update is done, potentially asynchronously, update the history list and browser UI

#### Example: single-page app "redirects"

A common scenario in web applications with a client-side router is to perform a "redirect" to a login page if you try to access login-guarded information. The following is an example of how one could implement this using the `navigate` event:

```js
appHistory.addEventListener("navigate", e => {
  const url = new URL(e.destinationEntry.url);
  if (url.pathname === "/user-profile") {
    // Cancel the navigation:
    e.preventDefault();

    // Do another navigation to /login, which will fire a new `navigate` event:
    location.href = "/login";
  }
});
```

_TODO: should these be combined into a helper method like `e.redirect("/login")`?_

In practice, this might be hidden behind a full router framework, e.g. the Angular framework has a notion of [route guards](https://angular.io/guide/router#preventing-unauthorized-access). Then, the framework would be the one listening to the `navigate` event, looping through its list of registered route guards to figure out the appropriate reaction.

NOTE: if you combine this example with the previous one, it's important that this route guard event handler be installed before the general single-page navigation event handler. Additionally, you'd want to either insert a call to `e.stopImmediatePropagation()` in this example, or a check of `e.defaultPrevented` in that example, to stop the other `navigate` event handler from proceeding with the canceled navigation. In practice, we expect there to be one large application- or framework-level `navigate` event handler, which would take care of ensuring that route guards happen before the other parts of the router logic, and preventing that logic from executing.

#### Example: affiliate links

A common [query](https://stackoverflow.com/q/11798336/3191) is how to append affiliate IDs onto links. Although this can be done server-side, sometimes it is convenient to do so client side, especially in the case of dynamic content. Today, this requires intercepting `click` events on `<a>` elements, or using a `MutationObserver` to watch for new link insertions. The `navigate` event provides a simpler way to do this:

```js
appHistory.addEventListener("navigate", e => {
  const url = new URL(e.destinationEntry.url);
  if (url.hostname === "store.example.com") {
    url.queryParams.set("affiliateId", "ead21623-781e-442f-a2c4-6cc1b2a9fda2");

    e.preventDefault();
    location.href = url;
  }
});
```

_TODO: it feels like this should be less disruptive than a cancel-and-perform-new-navigation; it's just a tweak to the outgoing navigation. Using the same code as the previous example feels wrong. Some brainstorming needed._

### Queued up single-page navigations

Consider trying to code a "next" button that performs a single-page navigation. This can be prone to race conditions if navigations are not instant. For example, if you're on `/photos/1` and click the next button twice, you should end up at `photos/3`, even if `photos/2` takes a long time to load and the click handler executes while the URL bar still reads `/photos/1`.

Concretely, code such as the following is buggy:

```js
let currentPhoto = 1;

document.querySelector("#next").onclick = async () => {
  await appHistory.pushNewEntry({ url: `/photos/${currentPhoto + 1}` });
};

appHistory.addEventListener("navigate", e => {
  const photoNumber = photoNumberFromURL(e.destinationEntry.url);

  if (photoNumber) {
    e.respondWith((async () => {
      const blob = await (await fetch(`/raw-photos/${photoNumber}.jpg`)).blob();
      const url = URL.createObjectURL(blob);
      document.querySelector("#current-photo").src = url;

      currentPhoto = photoNumber;
    })());
  }
});

function photoNumberFromURL(url) {
  const result = /\/photos/(\d+)/.exec((new URL(url)).pathname);
  if (result) {
    return Number(result[1]);
  }
  return null;
}
```

To fix this, the `appHistory.pushNewEntry()` and `appHistory.updateCurrentEntry()` APIs have callback variants. The callback will only be called after all ongoing navigations have finished. This allows non-buggy code such as the following:

```js
document.querySelector("#next").onclick = async () => {
  await appHistory.pushNewEntry(() => {
    const photoNumber = photoNumberFromURL(appHistory.currentEntry.url);
    return { url: `/photos/${photoNumber + 1}` };
  });
};
```

Although not shown in the above example, the callback could also return a `state` value.

_TODO: should the callback be able to say "nevermind, I don't care anymore, please don't navigate"? We could let the *caller* do that by passing an `AbortSignal` after the callback... Needs thinking about in the general context of multiple navigations ongoing._

In general, the idea of these callback variants is that there are cases where the new URL or state is not determined synchronously, and is a function of the current state of the world at the time the navigation is ready to be performed.

### Per-entry events

Each `AppHistoryEntry` has a series of events which the application can react to. **We expect these to mostly be used by decentralized parts of the application's codebase, such as components, to synchronize their state with the history list.** Unlike the `navigate` event, these events are not cancelable. They are used only for reacting to state changes, not intercepting or preventing navigations.

The application can use the `navigateto` and `navigatefrom` events to update the UI in response to a given entry becoming the current app history entry. For example, consider a photo gallery application. One way of implementing this would be to store metadata about the photo in the corresponding `AppHistoryEntry`'s `state` property. This might look something like this:

```js
async function showPhoto(photoId) {
  // In our app, the `navigate` handler will take care of actually showing the photo and updating the content area.
  await appHistory.pushNewEntry({ url: `/photos/${photoId}`, state: {
    dateTaken: null,
    caption: null
  } });

  // When we navigate away from this photo, save any changes the user made.
  appHistory.currentEntry.addEventListener("navigatefrom", e => {
    e.target.state.dateTaken = document.querySelector("#photo-container > .date-taken").value;
    e.target.state.caption = document.querySelector("#photo-container > .caption").value;
  });

  // If we ever navigate back to this photo, e.g. using the browser back button or
  // appHistory.navigateTo(), restore the input values.
  appHistory.currentEntry.addEventListener("navigateto", e => {
    document.querySelector("#photo-container > .date-taken").value = e.target.state.dateTaken;
    document.querySelector("#photo-container > .caption").value = e.target.state.caption;
  });
}
```

Note how in the event handler for these events, `event.target` is the relevant `AppHistoryEntry`, so that the event handler can use its properties (like `state`, `key`, or `url`) as needed.

Additionally, there's a `dispose` event, which occurs when an app history entry is permanently evicted and unreachable: for example, in the following scenario.

```js
const startingKey = appHistory.currentEntry.key;

appHistory.pushNewState();
appHistory.currentEntry.addEventListener("dispose", () => console.log(1));

appHistory.pushNewState();
appHistory.currentEntry.addEventListener("dispose", () => console.log(2));

appHistory.pushNewState();
appHistory.currentEntry.addEventListener("dispose", () => console.log(3));

appHistory.navigateTo(startingKey);
appHistory.pushNewState();

// Logs 1, 2, 3 as that branch of the tree gets pruned.
```

This can be useful for cleaning up any information in secondary stores, such as `sessionStorage` or caches, when we're guaranteed to never reach those particular history entries again.

### Current entry change monitoring

The `window.appHistory` object has an event, `currententrychange`, which allows the application to react to any updates to the `appHistory.currentEntry` property. This includes both navigations that change its value, and calls to `appHistory.updateCurrentEntry()` that change its `state` or `url` properties. This cannot be intercepted or canceled, as it occurs after the navigation has already happened; it's just an after-the-fact notification.

This event has one special property, `event.navigationStartTimeStamp`, which for same-document navigations gives the value of `event.timeStamp` for the corresponding `navigate` event. This allows it to be used for determining the overall load time, including the time it took for a promise passed to `e.respondWith()` to settle:

```js
appHistory.addEventListener("currententrychange", e => {
  if (e.navigationStartTimeStamp) {
    const loadTime = e.timeStamp - e.navigationStartTimeStamp;

    document.querySelector("#status-bar").textContent = `Loaded in ${loadTime} ms!`;
    analyticsPackage.sendEvent("single-page-app-nav", { loadTime });
  } else {
    document.querySelector("#status-bar").textContent = `Welcome to this document!`;
  }
});
```

_TODO: could we populate this for cross-document navigations too? Then it kind of overlaps with existing timing APIs, and is probably a lot harder to implement..._

_TODO: Add a non-analytics examples._

### Complete event sequence

Between the per-`AppHistoryEntry` events and the `window.appHistory` events, there's a lot of events floating around. Here's how they all come together for a typical same-document navigation (e.g. a fragment navigation, or a navigation initiated by `appHistory.pushNewEntry()`):

1. `window.appHistory.currentEntry` fires `navigatefrom`.
1. `window.appHistory` fires `navigate`. (Cancelable/`respondWith()`-able)
1. If the event is canceled:
    1. If this whole process was initiated by a call to `appHistory.navigateTo()`, `appHistory.back()`, or `appHistory.forward()`, fulfill that promise with `undefined`.
    1. Return.
1. If `navigateEvent.respondWith()` is called with a rejected promise:
    1. If this whole process was initiated by a call to `appHistory.navigateTo()`, `appHistory.back()`, or `appHistory.forward()`, reject that promise with the same rejection reason.
    1. Return.
1. After the promise argument to `navigateEvent.respondWith()` fulfills, or after one microtask if `respondWith()` is not called:
    1. `window.appHistory.currentEntry` is updated to its new value.
    1. `window.appHistory.currentEntry` fires `navigateto`.
    1. `window.appHistory` fires `currententrychange`.
    1. Any `AppHistoryEntry` instances which are now unreachable fire `dispose` events.
    1. If this whole process was initiated by a call to `appHistory.navigateTo()`, `appHistory.back()`, or `appHistory.forward()`, fulfill that promise with `undefined`.

For a cross-document navigation, the sequence is very similar, except if `navigateEvent.respondWith()` is not called, then we indeed proceed to the destination document, and steps (5.i)–(5.iii) happen in that destination document. (Step (5.iv) is not applicable, and step (5.v) does not happen at all since the promise has gone away, along with the old document.) Note that this destination document could be being restored from the browser's back/forward cache, in which case these events happen after the `pageshow` event, or the destination document could be created from scratch, in which case these events happen after the `DOMContentLoaded` event. _TODO not so sure about that last part._

## Guide for migrating from the existing history API

For web developers using the API, here's a guide to explain how you would replace usage of `window.history` with `window.appHistory`.

### Performing navigations

Instead of using `history.pushState(state, uselessTitle, url)`, use `await appHistory.pushNewEntry({ state, url })`.

Instead of using `history.replaceState(state, uselessTitle, url)`, use `await appHistory.updateCurrentEntry({ state, url })`. Note that if you omit the state value, i.e. if you say `appHistory.updateCurrentEntry({ url })`, then unlike `history.replaceState()`, this will copy over the current entry's state.

Instead of using `history.back()` and `history.forward()`, use `await appHistory.back()` and `await appHistory.forward()`. Note that unlike the `history` APIs, the `appHistory` APIs will ignore other frames, and will only control the navigation of your frame. This means it might move through multiple entries in the joint session history, skipping over any entries that were generated purely by other-frame navigations.

Additionally, for same-document navigations, you can test whether the navigation had an effect using a pattern like the following:

```js
const startingEntry = appHistory.currentEntry;
await appHistory.back();
if (startingEntry === appHistory.currentEntry) {
  console.log("We weren't able to go back, because there was nothing previous in the app history list");
}
```

Instead of using `history.go(offset)`, use `await appHistory.navigateTo(key)` to navigate to a specific entry. As with `back()` and `forward()`, `appHistory.navigateTo()` will ignore other frames, and will only control the navigation of your frame. If you specifically want to reproduce the pattern of navigating by an offset (not recommended), you can use code such as the following:

```js
const offsetIndex = appHistory.entries.indexOf(appHistory.currentEntry) + offset;
const entry = appHistory.entries[offsetIndex];
if (entry) {
  await appHistory.navigateTo(entry.key);
}
```

### Using `navigate` handlers plus non-history APIs

Many cases which use `history.pushState()` today can be replaced with `location.href`, or just deleted, when using `appHistory`. This is because if you have a listener for the `navigate` event on `appHistory`, that listener can use `event.respondWith()` to transform navigations that would normally be new-document navigations into same-document navigations. So for example, instead of

```html
<a href="/about">About us</a>
<button onclick="doStuff()">Do stuff</a>

<script>
window.doStuff = async () => {
  await doTheStuff();
  document.querySelector("main").innerHTML = await loadContentFor("/success-page");
  history.pushState(undefined, undefined, "/success-page");
};

document.addEventListener("click", async e => {
  if (e.target.localName === "a" && shouldBeSinglePageNav(e.target.href)) {
    e.preventDefault();
    document.querySelector("main").innerHTML = await loadContentFor(e.target.href);
    history.pushState(undefined, undefined, e.target.href);
  }
});
</script>
```

you could instead use a `navigate` handler like so:

```html
<a href="/about">About us</a>
<button onclick="doStuff()">Do stuff</a>

<script>
window.doStuff = async () => {
  await doTheStuff();
  location.href = "/success-page";
};

document.addEventListener("navigate", e => {
  if (shouldBeSinglePageNav(e.destinationEntry.url)) {
    e.respondWith((async () => {
      document.querySelector("main").innerHTML = await loadContentFor(e.destinationEntry.url);
    })());
  }
});
</script>
```

Note how in this case we don't need to use `appHistory.pushNewEntry()`, even though the original code used `history.pushState()`.

_TODO: we could also consider removing `appHistory.pushNewEntry()` if we're not sure about remaining use cases? Then we'd likely have to add a state argument to something like `location.assign()`..._

### Attaching and using history state

To update the current entry's state, instead of using `history.replaceState(newState)`, use `appHistory.updateCurrentEntry({ newState })`.

To read the current entry's state, instead of using `history.state`, use `appHistory.currentEntry.state`.

In general, state in app history is expected to be more useful than state in the `window.history` API, because:

- It can be introspected even for the non-current entry, e.g. using `appHistory.entries[i].state`.
- It is not erased by navigations that are not under the developer's control, such as fragment navigations (for which the state is copied over) and iframe navigations (which don't affect the app history list).

This means that the patterns that are often necessary to reliably store application and UI state with `window.history`, such as maintaining a side-table or using `sessionStorage`, should not be necessary with `window.appHistory`.

### Introspecting the history list

To see how many history entries are in the app history list, use `appHistory.entries.length`, instead of `history.length`. However, note that the semantics are different: app history entries only include same-origin contiguous entries for the current frame, and so that this doesn't reflect the history before the user arrived at the current origin, or the history of iframes. We believe this will be more useful for the patterns that people want in practice, such as showing an in-application back button if `appHistory.entries.length > 0`.

The app history API allows introspecting all entries in the app history list, using `appHistory.entries`. This should replace some of the workarounds people use today with the `window.history` API for getting a sense of the history list, e.g. as described in [whatwg/html#2710](https://github.com/whatwg/html/issues/2710).

Finally, note that `history.length` is highly non-interoperable today, in part due to the complexity of the joint session history model, and in part due to historical baggage. `appHistory`'s less complex model, and the fact that it will be developed in the modern era when there's a high focus on ensuring interoperability through web platform tests, means that using it should allow developers to avoid cross-browser issues with `history.length`.

### Watching for navigations

Today there are two events related to navigations, `hashchange` and `popstate`, both on `Window`. These events are quite problematic and hard to use; see, for example, [whatwg/html#5562](https://github.com/whatwg/html/issues/5562) or other [open issues](https://github.com/whatwg/html/issues?q=is%3Aissue+is%3Aopen+popstate) for some discussion. MDN's fourteen-step guide to ["When popstate is sent"](https://github.com/whatwg/html/issues?q=is%3Aissue+is%3Aopen+popstate) is also indicative of the problem.

The app history API provides several replacements that subsume these events:

- To react to and potentially intercept navigations before they complete, use the `navigate` event on `appHistory`. See the [Navigation monitoring and interception](#navigation-monitoring-and-interception) section for more details, including how the event object provides useful information that can be used to distinguish different types of navigations.

- To react to navigations that have completed, use the `currententrychange` event on `appHistory`. See the [Current entry change monitoring](#current-entry-change-monitoring) section for more details, including an example of how to use it to determine how long a same-document navigation took.

- To watch a particular entry to see when it's navigated to, navigated from, or becomes unreachable, use that `AppHistoryEntry`'s `navigateto`, `navigatefrom`, and `dispose` events. See the [Per-entry events](#per-entry-events) section for more details.

## Integration with the existing history API and spec

An `AppHistoryEntry` corresponds directly to a [session history entry](https://html.spec.whatwg.org/#session-history-entry) from the existing HTML specification. However, not every session history entry would have a corresponding `AppHistoryEntry`: `AppHistoryEntry` objects only exist for session history entries which are same-origin to the current one, and contiguous.

Example: if a browsing context contains history entries with the URLs

1. `https://example.com/foo`
1. `https://example.com/bar`
1. `https://other.example.com/whatever`
1. `https://example.com/baz`

then, if the current entry is (4), there would only be one `AppHistoryEntry` in `appHistory.entries`, corresponding to (4) itself. If the current entry is (2), then there would be two `AppHistoryEntries` in `appHistory.entries`, corresponding to (1) and (2).

Furthermore, unlike the view of history presented by `window.history`, `window.appHistory` only gives a view onto session history entries for the current browsing context; it does not present the joint session history, i.e. it is not impacted by frames.

To make this correspondence work, every spec-level session history entry would gain two new fields:

- key, containing a browser-generated UUID. This is what backs `appHistoryEntry.key`.
- app history state, containing a JavaScript value. This is what backs `appHistoryEntry.state`.

Note that the "app history state" field has no interaction with the existing "serialized state" field, which is what backs `history.state`. This route was chosen for a few reasons:

- The desired semantics of `AppHistoryEntry` state is that it be carried over on fragment navigations, whereas `history.state` is not carried over. (This is a hard blocker.)
- A clean separation can help when a page contains code that uses both `window.history` and `window.appHistory`. That is, it's convenient that existing code using `window.history` does not inadvertently mess with new code that does state management using `window.appHistory`.
- Today, the serialized state of a session history entry is only exposed when that entry is the current one. The app history API exposes `appHistoryEntry.state` for all entries, via `appHistory.entries[i].state`. This is not a security issue since all app history entries are same-origin contiguous, but if we exposed the serialized state value even for non-current entries, it might break some assumptions of existing code.
- We're still having some discussions about making `appHistoryEntry.state` into something more structured, like a key/value store instead of an arbitrary JavaScript value. If we did that, using a new field would be better, so that the structure couldn't be destroyed by code that does `history.state = "a string"` or similar.

Apart from these new fields, the session history entries which correspond to `AppHistoryEntry` objects will continue to manage other fields like document, scroll restoration mode, scroll position data, and persisted user state behind the scenes, in the usual way. The serialized state, title, and browsing context name fields would continue to work if they were set or accessed via the usual APIs, but they don't have any manifestation inside the app history APIs, and will be left as null by applications that avoid `window.history` and `window.name`.

_TODO: actually, we should probably expose scroll restoration mode, like `history.scrollRestoration`? That API has legitimate use cases, and we'd like to allow people to never touch `window.history`..._

Finally, all the higher-level mechanisms of session history entry management, such as the interaction with navigation, continue to work as they did before; the correspondence to `AppHistoryEntry` APIs does not change the processing there.

## Impact on back button and user agent UI

The app history API doesn't change anything about how user agents implement their UI: it's really about developer-facing affordances. Users still care about the joint session history, and so that will continue to be presented in UI surfaces like holding down the back button. Similarly, pressing the back button will continue to navigate through the joint session history, potentially across origins and out of the current app history (into a new app history, on the new origin).

An important consequence of this is that when iframes are involved, the back button may navigate through the joint session history, without changing the current _app history_ entry. For example, consider the following sequence:

1. `https://example.com/start` loads
1. The user navigates to `https://example.com/outer` by clicking a link. This page contains an iframe with `https://example.com/inner-start`.
1. The iframe navigates to `https://example.com/inner-end`.

The app history list for the outer frame contains two entries:

```
1. https://example.com/start
2. https://example.com/outer
```

The joint session session history contains three entries:

```
A. https://example.com/start
B. https://example.com/outer
   ┗ https://example.com/inner-start
C. https://example.com/outer
   ┗ https://example.com/inner-end
```

The user's back button, as well as the `history.back()` API, will navigate the joint session history back to (B). However, they will have no effect on the app history list; that will stay on (2). Pressing the back button or calling `history.back()` a second time would then move the joint session history back to (A), and the app history list back to (1).

Finally, note that user agents can continue to refine their mapping of UI to joint session history to give a better experience. For example, in some cases user agents today have the back button skip joint session history entries which were created without user interaction. We expect this heuristic would continue to be applied for `appHistory.pushNewEntry()`, just like it is for today's `history.pushState()`.


## Security and privacy considerations

Privacy-wise, this feature is neutral, due to its strict same-origin contiguous entry scoping. That is, it only exposes information which the application already has access to, just in a more convenient form. The storage of state in the `AppHistoryEntry`'s `state` property is a convenience with no new privacy concerns, since that state is only accessible same-origin; that is, it provides the same power as something like `sessionStorage`.

Security-wise, this feature does not touch on any security-sensitive areas of the browser or application code.

_TODO: W3C TAG security and privacy questionnaire._

## Stakeholder feedback

- W3C TAG: TODO file for TAG early design review
- Browser engines:
  - Chromium: No feedback so far
  - Gecko: No feedback so far
  - WebKit: No feedback so far
- Web developers: TODO

## Acknowledgments

This proposal is based on [an earlier revision](https://github.com/slightlyoff/history_api/blob/55b1d7a933cfd2dccddb16668b13dbc9ff06a3f2/navigator.md) by [@tbondwilkinson](https://github.com/tbondwilkinson), which outlined all the same capabilities in a different form. It also incorporates the ideas from [philipwalton@](https://github.com/philipwalton)'s [navigation event proposal](https://github.com/philipwalton/navigation-event-proposal/tree/aa0b688eab37f906660e60af8dc49df04d33c17f).

Thanks also to
[@chrishtr](https://github.com/chrishtr),
[@dvoytenko](https://github.com/dvoytenko),
[@housseindjirdeh](https://github.com/housseindjirdeh), and
[@slightlyoff](https://github.com/slightlyoff)
for their help in exploring this space and providing feedback.

## Appendix: full API surface, in Web IDL format

```webidl
partial interface Window {
  readonly attribute AppHistory appHistory;
};

[Exposed=Window]
interface AppHistory : EventTarget {
  readonly attribute AppHistoryEntry currentEntry;
  readonly attribute FrozenArray<AppHistoryEntry> entries;

  undefined updateCurrentEntry(optional AppHistoryEntryOptions options = {});
  undefined updateCurrentEntry(AppHistoryNavigationCallback);

  Promise<undefined> pushNewEntry(optional AppHistoryEntryOptions options = {});
  Promise<undefined> pushNewEntry(AppHistoryNavigationCallback callback);

  Promise<undefined> navigateTo(DOMString key);
  Promise<undefined> back();
  Promise<undefined> forward();

  readonly attribute EventHandler oncurrententrychange;
  readonly attribute EventHandler onnavigate;
};

[Exposed=Window]
interface AppHistoryEntry : EventTarget {
  readonly attribute DOMString key;
  readonly attribute USVString url;
  readonly attribute any state;

  readonly attribute EventHandler onnavigateto;
  readonly attribute EventHandler onnavigatefrom;
  readonly attribute EventHandler ondispose;
};

dictionary AppHistoryEntryOptions {
  USVString url;
  any state;
};

callback AppHistoryNavigationCallback = AppHistoryEntryOptions ();

[Exposed=Window]
interface AppHistoryNavigateEvent : Event {
  constructor(DOMString type, optional AppHistoryNavigateEventInit eventInit = {});

  readonly attribute boolean userInitiated;
  readonly attribute boolean sameOrigin;
  readonly attribute AppHistoryEntry destinationEntry;
  readonly attribute Element? fragmentTarget;
  readonly attribute FormData? formData;
};

dictionary AppHistoryNavigateEventInit : EventInit {
  boolean userInitiated = false;
  boolean sameOrigin = false;
  required AppHistoryEntry destinationEntry;
  FormData? formData = null;
};

[Exposed=Window]
interface AppHistoryCurrentEntryChangeEvent : Event {
  constructor(DOMString type, optional AppHistoryCurrentEntryChangeEventInit eventInit = {});

  readonly attribute DOMHighResTimeStamp? navigationStartTimeStamp;
};

dictionary AppHistoryCurrentEntryChangeEventInit : EventInit {
  DOMHighResTimeStamp? navigationStartTimeStamp = null;
};
```

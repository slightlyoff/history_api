# Modal Close Signals

_Note: the authors of the proposal are strongly considering an [alternative proposal](#bundling-this-with-high-level-apis) of bundling this with new higher-level APIs for modals, instead of separating out the close signal problem specifically. So keep in mind that this proposal is extra-tentative, as you continue reading!_

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of contents

- [The problem](#the-problem)
- [Goals](#goals)
- [What developers are doing today](#what-developers-are-doing-today)
- [Proposal](#proposal)
  - [Asking for confirmation](#asking-for-confirmation)
  - [User activation gating](#user-activation-gating)
  - [Signaling close yourself](#signaling-close-yourself)
  - [Platform implementation notes](#platform-implementation-notes)
    - [Android: interaction with fragment navigation](#android-interaction-with-fragment-navigation)
    - [iOS: do nothing](#ios-do-nothing)
    - [Keyboards attached to mobile devices](#keyboards-attached-to-mobile-devices)
  - [Abuse analysis](#abuse-analysis)
  - [Realistic examples](#realistic-examples)
    - [A sidebar](#a-sidebar)
    - [A picker](#a-picker)
    - [A custom dialog](#a-custom-dialog)
- [Bonus proposals](#bonus-proposals)
  - [Extending `<dialog>`](#extending-dialog)
  - [Unifying existing close signals](#unifying-existing-close-signals)
- [Alternatives considered](#alternatives-considered)
  - [Integration with the history API](#integration-with-the-history-api)
  - [Automatically translating all close signals to <kbd>Esc</kbd>](#automatically-translating-all-close-signals-to-kbdesckbd)
  - [Not gating on transient user activation](#not-gating-on-transient-user-activation)
  - [Bundling this with high-level APIs](#bundling-this-with-high-level-apis)
- [Security and privacy considerations](#security-and-privacy-considerations)
- [Stakeholder feedback](#stakeholder-feedback)
- [Acknowledgments](#acknowledgments)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## The problem

Modals are UI components that are layered on top of all other content and take interaction focus. Some examples are:

- a `<dialog>` element, especially the [`showModal()` API](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement/showModal);
- a sidebar menu;
- a lightbox;
- a custom picker input (e.g. date picker);
- a custom context menu;
- fullscreen mode.

An important common feature of these modals is that they are designed to be easy to close, with a uniform interaction mechanism for doing so. Typically, this is the <kbd>Esc</kbd> key on desktop platforms, and the back button on some mobile platforms (notably Android). Game consoles also tend to use a specific button as their "close/back" button. Some platforms have no uniform close signals; to our knowledge, iOS falls in this category.

_TODO: do assistive technologies have their own close signals? On desktop it's probably just the <kbd>Esc</kbd> key, but maybe on mobile?_

We define a **close signal** as a platform-mediated interaction that's intended to close a modal. This is distinct from page-mediated interactions, such as clicking on an "x" or "Done" button, or clicking on the backdrop outside of the modal.

Currently, web developers have no good way to handle these close signals for their own modals. This is especially problematic on Android devices, where the back button is the traditional close signal. Imagine a user filling in a twenty-field form, with the last item being a custom date picker modal. The user might click the back button hoping to close the date picker, like they would in a native app. But instead, the back button navigates the web page's history tree, likely closing the whole form and losing the filled information.

This explainer proposes a new API to enable web developers, especially component authors, to better handle these close signals.

## Goals

Our primary goals are as follows:

- Allow web developers to intercept modal close signals (e.g. <kbd>Esc</kbd> on desktop, back button on Android) to close custom modals that they create.

- Discourage platform-specific code for handling modal close signals, by providing an API that abstracts away the differences.

- Allow future platforms to introduce new modal close signals (e.g., a "swipe away" gesture), such that code written against the API proposed here automatically works on such platforms.

- Allow the developer to confirm a close signal, in a similar fashion to `beforeunload`, to avoid potential data loss.

- Prevent abuse that traps the user on a given history state by disabling the back button's ability to actually navigate backward in history.

- Be usable by component authors, in particular by avoiding any global state modifications that might interfere with the app or with other components.

The following are nice-to-have goals:

- Specify uniform-per-platform behavior across existing platform modals, e.g. `<dialog>`, `<input>` pickers, and fullscreen. (E.g., currently the back button on Android does not close modal `<dialog>`s, but instead navigates history. It does close `<input>` pickers and cancel fullscreen.)

- Allow web developers to confirm the close signal for not just custom modals, but also `<dialog>`s, so that the user can choose to keep the dialog open. (Currently <kbd>Esc</kbd> on desktop always closes modal `<dialog>`s, with no opportunity to confirm this action with the user.)

The following is a goal we wish we could meet, but don't believe is possible to meet while also achieving our primary goals:

- Avoid an awkward transition period, where the Android back button closes modals on sites that adopt this new API, but navigates history on sites that haven't adopted it. In particular, right now users generally know that the Android back button fails to close modals in web apps, and avoid it when modals are open. We worry about this API causing a state where users are no longer sure which action it performs.

## What developers are doing today

On desktop platforms, this problem is currently solved by listening for the `keydown` event, and closing the modal when the <kbd>Esc</kbd> key is pressed. Built-in platform APIs, such as [`<dialog>`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement/showModal), [fullscreen](https://developer.mozilla.org/en-US/docs/Web/API/Fullscreen_API), or [`<input type="date">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/date), will do this automatically. Note that this can be easy to get wrong, e.g. by accidentally listening for `keyup` or `keypress`.

Additionally, note that the inability to prevent closing of `<dialog>`s via <kbd>Esc</kbd> makes them hard to use for some web developers, and requires them to roll their own custom dialog modals. _TODO: evidence of this would be good._

On mobile platforms, getting the right behavior is significantly harder. First, platform-specific code is required, to do nothing on iOS and to capture the back button on Android. Next, capturing the back button on Android requires manipulating the history stack using the [history API](https://developer.mozilla.org/en-US/docs/Web/API/History_API). This is a poor fit for several reasons:

- This UI state is non-navigational, i.e., the URL doesn't and shouldn't change when opening a modal. When the page is reloaded or shared, web developers generally don't want to automatically re-open the same modal.

- It's very hard to determine when the modal state is popped out of history, because both moving forward and backward through history emit the same `popstate` event, and that event can be emitted for history navigation that traverses more than one entry at a time.

- The `history.state` API, used to store the modal state, is rather fragile: user-initiated fragment navigation, as well as other calls to `history.pushState()` and `history.replaceState()`, can override it.

- History navigation is not cancelable. Implementing "Are you sure you want to close this dialog without saving?" requires letting the history navigation complete, then potentially re-establishing the modal state if the user declines to close the modal.

- A shared component that attempts to use the history API to implement these techniques can easily corrupt a web application's router.

## Proposal

The proposal is to introduce a new API, the `ModalCloseWatcher` class, which has the following basic API:

```js
// Note: constructor could throw a "NotAllowedError" DOMException, if used too
// many times without user interaction.
const watcher = new ModalCloseWatcher();

// This fires when the user sends a close signal, e.g. by pressing Esc on
// desktop or by pressing Android's back button.
watcher.onclose = () => {
  myModal.close();
};

// You should destroy watchers which are no longer needed, e.g. if the
// modal closes normally. This will prevent future events on this watcher.
myModalCloseButton.onclick = () => {
  watcher.destroy();
  myModal.close();
};
```

If more than one `ModalCloseWatcher` is active at a given time, then only the most-recently-constructed one gets events delivered to it. A watcher becomes inactive after a `close` event is delivered, or the watcher is explicitly `destroy()`ed.

Read on for more details and realistic usage examples.

### Asking for confirmation

There's a more advanced part of this API, which allows asking the user for confirmation if they really want to close the modal. This is useful in cases where, e.g., the modal contains a form with unsaved data. The usage looks like the following:

```js
watcher.onbeforeclose = e => {
  if (hasUnsavedData) {
    e.preventDefault();
  }
};
```

If the event is canceled, then the browser will show some UI saying something like "Close modal? Changes you made may not be saved." For [abuse prevention](#abuse-analysis), this text is _not_ configurable.

Furthermore, unlike `Window`'s `beforeunload` event, this browser UI does _not_ block the event loop. Instead, it simply has the effect of delaying the `close` event until the user says "Yes".

If the user says "No" (i.e., keep the modal open), then the `ModalCloseWatcher` remains active, and can receive future `beforeclose` (and `close`) events. But once the user says "Yes" (i.e., confirms closing the modal), the `ModalCloseWatcher` receives the final `close` event, and becomes inactive.

The exact UI for this confirmation is up to the user agent. It could be a `beforeunload`-style modal dialog, but the resulting "modal on top of a modal" might be ugly. _TODO maybe we can get some mock-ups of better ideas._

Note that the `beforeclose` event is not fired when the user navigates away from the page: i.e., it has no overlap with `beforeunload`. `beforeunload` remains the best way to confirm a page unload, with `beforeclose` only used for confirming a modal close signal.

### User activation gating

It was mentioned above that the `new ModalCloseWatcher()` constructor can throw if called too many times without user activation. Specifically, the design is that the page gets one "free" active `ModalCloseWatcher` at a time. After that, any further `ModalCloseWatcher` constructions require [transient activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation-gated-api), i.e., the construction must happen as part of or shortly after a user interaction event like `click`, `pointerup`, or `keydown`.

The motivation for this is that, for platforms like Android where the modal close gesture is to use the back button, we need to prevent abuse that traps the user on a page by effectively disabling their back button. This restriction means that the back button can only be intercepted as many times as user activation was given to the document, plus one.

The allowance for a single non-activation-triggered `ModalCloseWatcher` is to allow use cases like "session inactivity timeout" modals, or high-priority interrupt modals. The page can create a single one of these at a given time, which we believe strikes a good balance between meeting realistic use cases and preventing abuse.

Note that for developers, this means that calling `modalCloseWatcher.destroy()` properly is important, as doing so will free up the "free `ModalCloseWatcher` slot", if it has been previously consumed.

### Signaling close yourself

The API has an additional convenience method, `watcher.signalClosed()`, which acts _as if_ a close signal had been sent by the user. The intended use case is to allow centralizing close-handling code. So the above example of

```js
watcher.onclose = () => myModal.close();

myModalCloseButton.onclick = () => {
  watcher.destroy();
  myModal.close();
};
```

could be replaced by

```js
watcher.onclose = () => myModal.close();

myModalCloseButton.onclick = () => watcher.signalClosed();
```

deduplicating the `myModal.close()` call by having the developer put all their close-handling logic into the watcher's `close` event handler.

If called from within [transient user activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation-gated-api), `signalClosed()` also invokes `beforeclose` event handlers, and will potentially cause a confirmation UI to come up which could delay the `close` event. If called without user activation, then it skips straight to the `close` event.

In either case, as usual, reaching the `close` event will inactivate the `ModalCloseWatcher`, meaning it receives no further events in the future and it no longer occupies the "free `ModalCloseWatcher` slot", if it was previously doing so.

### Platform implementation notes

#### Android: interaction with fragment navigation

Consider a modal which contains a lot of text, such as a terms of service with a table of contents, or a preview overlay that shows an interactive version of this very document. Within such a modal, it'd be possible to perform fragment navigation. Although these situations are hopefully rare in practice, we need to figure out how they impact the proposed API.

In particular, the problem case is again Android, where the back button serves as both a potential close signal and a way of navigating the history stack. We suggest that for Android, the back button still acts as a modal close signal whenever any modal close watcher is active. That is, it should only do fragment navigation back through the history stack when there are no active modal close watchers, even if fragment navigation has occurred since the modal was shown.

This does mean that the modal could be closed, while the page's current fragment and its history stack still reflect fragment navigations that were performed inside the modal. The application would need to clean up such history entries to give a good user experience. We believe this is best done through a history-specific API, e.g. the existing `window.history` API or one of the better ones discussed in [slightlyoff/history_api](https://github.com/slightlyoff/history_api), since this sort of batching and reversion of history entries is a generic problem and not modal-specific.

The alternative is for implementations to treat back button presses as fragment navigations, only treating them as modal close signals once the user has returned to the same history entry as the modal was opened in. But discussion with framework authors indicated this would be harder to deal with and reason about. In particular, this would introduce significant divergences between platforms: on Android, closing a modal would not require any history-stack-cleaning, whereas on other platforms it would. So, in accordance with our [goal](#goals) of discouraging platform-specific code, we encourage implementations to go with the model suggested above, which enables web developers to use the same cleanup code without platform-specific branches. (Or they can avoid the problem altogether, by not providing opportunities for fragment navigation while modals are opened.)

#### iOS: do nothing

On iOS, where to our knowledge there is no uniform close signal, this API is very simple. `ModalCloseWatcher` should still exist, and should still require transient user activation to construct, for interoperability. But, the `beforeclose` and `close` events never fire. Instead, modals will be closed normally via whatever specific interaction makes sense for the modal in question, such as clicking on the backdrop or swiping down.

Note that some application and component developers adjust to iOS's lack of close signals by making "Done" or "x" buttons more explicit in their UI. So while the browser implementation of `ModalCloseWatcher` would do nothing, components might end up doing something on their own.

#### Keyboards attached to mobile devices

TODO this might change the rest of the doc's discussion of iOS. https://osxdaily.com/2019/04/12/how-type-escape-key-ipad/

### Abuse analysis

As discussed [above](#user-activation-gating), for platforms like Android where the modal close gesture is to use the back button, we need to prevent abuse that traps the user on a page by effectively disabling their back button. The user activation gating is intended to combat that. Notably, that protection is already stronger than anything done today for the `history.pushState()` API, which is another means by which apps can attempt to trap the user on the page. [See discussion below](#not-gating-on-transient-user-activation) for more on that.

Additionally, we note that in most back button UIs, the user always has an escape hatch of holding down the back button and explicitly choosing a history step to navigate back to. This is never a modal close signal.

Another potential abuse vector is the confirmation dialogs triggered by the `beforeclose` event. We saw much abuse of `Window`'s `beforeunload` event, during the era where it allowed user-provided strings (via `event.returnValue` or the event handler return value). The issue there is putting web-developer-supplied strings into trusted browser UI. Thus, we have designed this API to avoid this; the only action the web developer can take is the binary signal of either canceling the event, or letting it proceed.

### Realistic examples

The above sections give illustrative usage of the API. The following ones show how the API could be incorporated into realistic apps and UI components.

#### A sidebar

For a sidebar (e.g. behind a hamburger menu), which wants to hide itself on a user-provided close signal, that could be hooked up as follows:

```js
const hamburgerMenuButton = document.querySelector('#hamburger-menu-button');
const sidebar = document.querySelector('#sidebar');

hamburgerMenuButton.addEventListener('click', () => {
  const watcher = new ModalCloseWatcher();

  sidebar.animate([{ transform: 'translateX(-200px)' }, { transform: 'translateX(0)' }]);

  watcher.onclose = () => {
    sidebar.animate([{ transform: 'translateX(0)' }, { transform: 'translateX(-200px)' }]);
  };

  // Close on clicks outside the sidebar.
  document.body.addEventListener('click', e => {
    if (e.target.closest('#sidebar') === null) {
      watcher.signalClose();
    }
  });
});
```

#### A picker

For a "picker" control that wants to close itself on a user-provided close signal, code like the following would work:

```js
class MyPicker extends HTMLElement {
  #button;
  #overlay;
  #watcher;

  constructor() {
    super();
    this.#button = /* ... */;

    this.#overlay = /* ... */;
    this.#overlay.hidden = true;
    this.#overlay.querySelector('.close-button').addEventListener('click', () => {
      this.#watcher.signalClose();
    });

    this.#button.onclick = () => {
      this.overlay.hidden = false;

      this.#watcher = new ModalCloseWatcher();
      this.#watcher.onclose = () => this.overlay.hidden = true;
    }
  }
}
```

_TODO: include a React example._

#### A custom dialog

For a dialog, which might contain unsaved data, code like the following would work:

```js
class MyCustomDialog extends HTMLElement {
  #watcher;
  #hasUnsavedData;

  get hasUnsavedData() {
    return this.#hasUnsavedData;
  }
  set hasUnsavedData(v) {
    this.#hasUnsavedData = Boolean(v);
  }

  showModal() {
    // Normal dialog implementation left to the reader.

    this.#setupWatcherIfPossible();
    this.addEventListener('close', () => this.#watcher.?destroy());
  }

  close() {
    // Normal dialog implementation left to the reader.
    this.dispatchEvent(new Event('close'));
  }

  #setupWatcherIfPossible() {
    try {
      #watcher = new ModalCloseWatcher();
    } catch {
      // The web developer using the component called showModal() outside of transient user activation.
      return;
    }

    #watcher.onbeforeclose = e => {
      if (this.#hasUnsavedData) {
        e.preventDefault();
      }
    };

    #watcher.onclose = () => {
      this.close();
    };
  }
}
```

_TODO: include a React example._

## Bonus proposals

With `ModalCloseWatcher` as a foundation, we can accomplish our nice-to-have goals as follows:

### Extending `<dialog>`

We could add a `beforeclose` even to `<dialog>` which behaves in the exact same way as the `beforeclose` event on a `ModalCloseWatcher`: it is fired during a modal close signal, and canceling it prevents the dialog from closing.

This provides an important new capability, of allowing web developers to double-check with the user if the user presses the <kbd>Esc</kbd> key, or otherwise sends a modal close signal. Currently, the browser will automatically close the `<dialog>`, potentially leading to data loss.

We would add the same restrictions on this event as `ModalCloseWatcher` has: it would only be fired if the corresponding call to `dialogEl.showModal()` was done during user activation. And with this integration, `<dialog>`s would potentially take up the "free `ModalCloseWatcher` slot", i.e. you couldn't both pop up a `<dialog>` without user activation, and then construct a new `ModalCloseWatcher` without user activation.

Similarly, `<dialog>`'s existing `close()` method would, if called from user activation, trigger the `beforeclose` event.

In general, we can think of this as if `<dialog>` is implemented under the hood using `ModalCloseWatcher`. Likely, it would be, in implementations and the specification. The web developer would not see this, however, since directly exposing it would be redundant with `<dialog>`'s existing functionality. That is, it would be awkward to exose something like `dialogEl.closeWatcher`, since it would lead to confusing situations like `dialogEl.closeWatcher.signalClose()` living alongside `dialogEl.close()`, or `dialogEl.closeWatcher.addEventListener("close", ...)` living alongside `dialogEl.addEventListener("close", ...)`.

### Unifying existing close signals

The specification for `ModalCloseWatcher` will have a specific definition for "modal close signal". Although the actual signal is implementation-defined and platform-dependent, having a definition gives us something we can reference to unify other specifications.

Specifically, we would update the following specifications:

- [Fullscreen](https://fullscreen.spec.whatwg.org/#ui) would mention that a modal close signal should act as an instruction to the user agent to exit fullscreen
- [`<dialog>`](https://html.spec.whatwg.org/multipage/interactive-elements.html#canceling-dialogs) would specify that modal close signals should run the dialog cancelation steps
  - This would require some subtlety to avoid [back button trapping abuse](#abuse-analysis). Probably the simplest solution is to ensure that, on platforms where the back button is the modal close signal, this clause only applies to dialogs where `showModal()` was called from transient user activation.
- Perhaps we'd update `<input>`'s specification as well, but it is much more vague than the other two about UI

These updates would encourage user agents to make these built-in features behave the same as each other, and behave the same as `ModalCloseWatcher`-based web-developer-created interfaces.

## Alternatives considered

### Integration with the history API

Because of how modals interact with the back button on Android, one might think it natural to make handling them part of either the existing history API, or a revised history API. This idea has some precedent in mobile application frameworks that integrate modals into their "back stack".

However, on the web the history API is intimately tied to navigations, the URL bar, and application state. Using it for UI state is generally not great. See also the [above discussion](#what-developers-are-doing-today) of how developers are forced to use the history API today for this purpose, and how poorly it works.

In fact, we're hopeful that by tackling this proposal separately from the history API, other efforts to improve the history API will be able to focus on actual navigations, instead of on modals.

### Automatically translating all close signals to <kbd>Esc</kbd>

If we assume that developers already know to handle the <kbd>Esc</kbd> key to close their modals, then we could potentially translate other close signals, like the Android back button, into <kbd>Esc</kbd> key presses. The hope is then that application and component developers wouldn't have to update their code at all: if they're doing the right think for that common desktop close signal, they would suddenly start doing the right thing on other platforms as well. This is especially attractive as it could help avoid the awkward transition period mentioned in the [goals](#goals) section.

However, upon reflection, such a solution doesn't really solve the Android back button problem. Given an Android back button press, the browser needs to know: should this navigate history, or be translated to an <kbd>Esc</kbd> key press? For custom modals, the only way to know is for the web developer to tell the browser that a modal is open. So our goal of requiring no code modifications, or awkward transition period, is impossible. Given this, the strangeness of synthesizing fake <kbd>Esc</kbd> keypresses does not have much to recommend it.

### Not gating on transient user activation

We're gating the creation of more than one `ModalCloseWatcher` on transient user activation as an [anti-abuse measure](#abuse-analysis). However, this protection is stronger than the existing protections against excessive `history.pushState()` use, which are are more vague and less mandatory in [that method's spec](https://html.spec.whatwg.org/multipage/history.html#dom-history-pushstate):

> Optionally, return. (For example, the user agent might disallow calls to these methods that are invoked on a timer, or from event listeners that are not triggered in response to a clear user action, or that are invoked in rapid succession.)

We could gate the creation of `ModalCloseWatcher` on similarly-vague and optional protections. This would allow more free use of it, especially on platforms that don't use the back button as a close signal and so don't need the abuse protection.

But on balance, we'd prefer to start with the transient user activation restriction (plus one "free" `ModalCloseWatcher`), mainly for reasons of interoperability. Allowing platforms to differ in when `ModalCloseWatcher`s can be created would potentially create a race to the bottom, where nobody can be stricter than the most widely-used implementation. Arguably, if we'd designed the history API today, `history.pushState()` itself would only be usable from transient user activation.

### Bundling this with high-level APIs

The proposal here exposes `ModalCloseWatcher` as a primitive. However, watching for close signals is only a small part of what makes modals difficult. There is also [top layer interaction](https://github.com/whatwg/html/issues/4633), [blocking interaction with the main document](https://github.com/whatwg/html/issues/897) (including trapping focus within the modal while it is open), providing appropriate accessibility semantics, and determining the appropriate screen position.

Instead of providing the individual building blocks for all of these pieces, it may be better to bundle them together into high-level semantic elements. We already have `<dialog>`; we could imagine others such as `<popup>`, `<toast>`, `<tooltip>`, etc. These would then bundle the appropriate features, e.g. while all of them would benefit from top layer interaction, `<toast>` and `<tooltip>` do not need to handle close signals.

This likely leads to a better and more uniform user experience. If developers use the correct high-level element, then they'll get the appropriate behavior, including close signal handling, for free. For example, not all developers realize that menus should respond to close signals. If we rely on them to hook up `ModalCloseWatcher` to their custom menu code, some developers might fail to do so. Whereas if modal close signal handling is automatic when using a `<popup type="menu">` element, it will just work.

If we went down this path, much of the research and semantics in this explainer document would still be useful. We'd essentially pivot the proposal so that all the appropriate high-level elements would get extended in the same way we're currently proposing [for `<dialog>`](#extending-dialog). That is, they'd have `beforeclose` and `close` events, and `close()` methods, that follow all the rules outlined in this proposal. They would all centrally reference the "close signal" concept, in specifications and implementations, to ensure uniformity. There might even be a `ModalCloseWatcher` class used internally by implementations! But, there wouldn't be a JavaScript-exposed `ModalCloseWatcher` API.

The authors of this proposal find this alternative quite attractive, and welcome any input to help weigh the JavaScript-exposed `ModalCloseWatcher` class versus only using it under the hood.

## Security and privacy considerations

This feature has no security considerations to speak of, as a purely DOM API.

Regarding privacy, this proposal should be slightly privacy-positive, as it masks platform-specific user interaction patterns behind a uniform cross-platform interface.

TODO: W3C TAG security and privacy questionnaire.

## Stakeholder feedback

- W3C TAG: TODO file for TAG early design review
- Browser engines:
  - Chromium: No feedback so far
  - Gecko: No feedback so far
  - WebKit: No feedback so far
- Web developers: TODO

## Acknowledgments

This proposal is based on [an earlier analysis](https://github.com/slightlyoff/history_api/blob/67c29f3bd7710f2adff737bd665cd9338a68b25d/history_and_modals.md) by [@dvoytenko](https://github.com/dvoytenko).

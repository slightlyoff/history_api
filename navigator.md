# Goals

This proposal aims to:

- Collect information about history into one place.
- Provide a unique identifier for history frames.
- Allow inspection of the history stack.
- Provide global listeners of history changes.
- Provide local listenenrs of changes to a specific history frame.
- Encourage developers to only update the part of the URL or state Object that they need.
- Provide a new API for navigating to a specific past history frame, rather than relying on `go()`.
- Provide Promise return values for all history mutating APIs.

This proposal creates a new API surface, `window.history.navigator`.

# Proposal and justifying use cases

## Creation of a `Frame` Object with a unique `key` identifier

This proposes creating a new `Frame` Object to collect information about a single history frame in the history stack.

```
class Frame {
  // Unique identifier for this Frame.
  readonly key: string;
  // URL of this Frame.
  readonly url: URL;
  // Params passed into the original Frame.
  readonly params: {string: Any};
}
```

This feature supports the following use cases:

- Collects all information about the history into a single place.
- Provides a URL Object, rather than a string, for the URL.
- Provides reading the specific `params` Object for the `Frame`, so that users can read the data they previously wrote to history.
- Provides a unique key. This provides a primary key for applications to store information that is related to this history frame, such as additional state that should be used for restoration. Applications have used the URL in the past as a primary key, but it is not guaranteed to be unique.

## History frame events

This proposes adding an additional method to `Frame`:

```
// focus - the frame navigated in.
// blur - the frame navigated out
// beforeRemove - the frame is about to be navigated out. Enables rejecting the removal of the frame under certain circumstances.
// evicted - the frame is no longer navigable, due to being in a branch of history that was evicted or became unreachable.
addListener('focus'|'blur'|'beforeRemove'|'evicted') {}
```

This features supports the following use cases:

- Evicting information stored in secondary stores (sessionStorage, e.g.) when a history frame is permanently evicted (no longer in stack).
- Enables updating UI in response to specific frames entering/removing the view.
- Enables preventing a specific frame from being navigated away from.

## Inspection of the history stack

This proposes adding two new methods to `window.history.navigator`:

```
// Returns the current Frame.
getFrame(): Frame {}

// Returns all Frames.
getStack(): {frames: []Frame {}, index: number}
```

This feature supports the following use cases:

- Creating (or hiding) a UI element that could allow users to choose which state they want to navigate back to.
- No longer need to store information from previous history frames globally, because it would be accessible at any time. Today applications duplicate the history state Object entirely in memory to prevent it from being removed. Even if the history state Object went out of scope, an application could look in the back stack to find it.
- No longer need to track history externally via popstate or other events, since applications could see for themself what is in their forward and back stack.
- Could expose a view of the history stack that does NOT include iframe history frames, which could become invisible to the hosting application.

## Global history change listener

This proposes adding a new method to `window.history.navigator`:

```
// frameChange - a new frame was pushed or replaced.
// navigation - the history stack changed, but no new frame was added or removed. Equivalent to popstate, but with a better name.
addListener('frameChange'|'navigation') {}
```

This feature supports the following use cases:

- Enable listening for new frames being pushed or replaced (today popstate only fires on navigations)
- Provides a navigation event that can be used to provide more information (user-initiated, back-forward) as well as provide user controls for preventing undesired navigations.

## Creating new history frames

This proposes adding a new method to `window.history.navigator`.

```
// Takes as its only argument a callback that updates the input url params.
// The return value is an Object with optional properties.
// url: the url parameter possibly modified or undefined, meaning to preserve the existing url.
// params: the params parameter possibly modified or undefined, meaning to preserve the existing params.
// replace: true/false/undefined, if true replaces the current Frame instead of pushing a new one.
createFrame(({url: URL, params: {string: Any}}):
    {url: URL|undefined, params: {string: Any}|undefined, replace: boolean|undefined} => {}): Promise<Frame> {}
```

This feature supports the following use cases:

- Supports pushState/replaceState-like behavior today but with a better API.
- Returns a Promise that callers can listen to when the Frame has been changed.
- Having a callback encourages users to mutate the url and params and return them.
-  Users no longer have to care about arguments they don't use. If they don't want to update the URL, they don't list it in their callback or return. If a value is not returned, the existing value is used (url or params).

## Navigate to specific `Frame`

This proposes creating a method on `window.history.navigator`:

```
// Navigate to the Frame, if it is in the stack.
navigateTo(key: string): Promise<Frame> {}
```

This feature supports the following use cases:

- Navigating by key rather than direction enables users to not care about other entries in the history stack.
- Returning a Promise enables users to wait for their URL change to be reflected.

## Promises for all history navigations

This proposes creating new methods on `window.history.navigator`:

```
  // Navigates back one Frame.
  back(): Promise<Frame> {}

  // Navigates forward one Frame.
  forward(): Promise<Frame> {}
```

This features supports the following use cases:

- back/forward navigations are asynchronous (URL changes not synchronously updated) but do not return Promises. This enables the application to wait for the history navigation to finish.
- Allows the application to get the current Frame in the resolved value after its finished.

# API summary

```
class Frame {
  // Unique identifier for this Frame.
  readonly key: string;
  // URL of this Frame.
  readonly url: URL;
  // Params passed into the original Frame.
  readonly params: {string: Any}|undefined;

  // focus - the frame navigated in.
  // blur - the frame navigated out
  // beforeRemove - the frame is about to be navigated out. Enables rejecting the removal of the frame under certain circumstances.
  // unreachable - the frame is no longer navigable, due to being in a branch of history that was evicted or became unreachable.
  addListener('focus'|'blur'|'beforeRemove'|'unreachable') {}
}

class Navigator {
  // Takes as its only argument a callback that updates the input url params.
  // The return value is an Object with optional properties.
  // url: the url parameter possibly modified or undefined, meaning to preserve the existing url.
  // params: the params parameter possibly modified or undefined, meaning to preserve the existing params.
  // replace: true/false/undefined, if true replaces the current Frame instead of pushing a new one.
  // The function rejects if the return value is not the exact same for both url and params.
  update(({url: URL, params: {string: Any}}):
      {url: URL|undefined, params: {string: Any}|undefined, replace: boolean|undefined} => {}): Promise<Frame> {}

  // Navigate to the Frame.
  navigateTo(key: string): Promise<Frame> {}

  // Returns the current Frame.
  getFrame(): Frame {}

  // Returns all Frames.
  getStack(): {frames: []Frame {}, index: number}

  // Navigates back one Frame.
  back(): Promise<Frame> {}

  // Navigates forward one Frame.
  forward(): Promise<Frame> {}

  // frameChange - a new frame was pushed or replaced.
  // navigation - the history stack changed, but no new frame was added or removed. equivalent to popstate, but with a better name.
  addListener('frameChange'|'navigation') {}
}
```

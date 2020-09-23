# Improved `history.state`

One of the places applications can store arbitrary data is in the `history.state` Object which can be provided when calling `history.pushState` and `history.replaceState`. The Object provided can be anything that can be serialized. The Object can be retrieved by calling `history.state`, and is preserved even after page reloads or navigations.

Use cases for `history.state`:
1. Many applications use the `history.state` Object to store UI data that should not be part of the public URL of the page (i.e., not sharable) but that should be avialable to a page when a user agent navigates back to a history entry or reloads the page. Applications use this like a session storage that is linked to the navigation model. This data is useful for restoring UI.
2. Some router libraries use `history.state` to add unique ids to history entries so that the application can better track its navigational model. In extreme cases, router libraries have attempted to serialize the entire history stack into `history.state` to provide better a navigational model to the application. This can be useful to have a better sense of what navigational path the user agent took to reach the current UI, which could inform UI display (e.g. showing a back button only if there is a previous entry) or navigational control (e.g. navigating backwards to a specific UI state)

The existing `history.state` has some drawbacks:
1. Clicks on fragment-changing links and calls to `location.replace` and `location.assign` overwrite the history.state Object with `null`. This means that the history state Object is either overwritten if the URL is replacing the existing URL, or it is lost to future history entries. This makes state restoration difficult or impossible when this occurs.
2. JavaScript calls to `history.replaceState`, `history.pushState` need to copy and extend the existing state Object. This means that the application needs to agree on a top-level structure for the history state. Most applications use an Object literal with keys to represent namespaces.

# Proposed API

A simple approach to improve this situation would be to create a secondary state API that provides a session-storage like API for updating the current history.state Object.

This API would be fairly straightforward:

```
// Akin to a call to history.replaceState(window.location.href, '', {foo: serializedUiState});
history.stateStorage.setItem('foo', serializedUiState);
// Returns serializedUiState
history.stateStorage.getItem('foo');

// Setting other items does not overwrite existing items
history.stateStorage.setItem('bar', otherSerializedUiState);
// Returns serializedUiState
history.stateStorage.getItem('foo');

// When navigating back, the state storage for foo is no longer available.
history.back();
// Returns undefined
history.stateStorage.getItem('foo');

// When navigating forward, the state storage for foo is available again.
history.forward();
// Returns serializedUiState
history.stateStorage.getItem('foo');

// When pushing a new URL, the browser copies the current value of 'foo' automatically.
history.pushState(newUrl, '', {});
// Returns serializedUiState
history.stateStorage.getItem('foo');

// Entries can be updated as well.
history.stateStorage.setItem('foo', newSerializedUiState);
// Returns newSerializedUiState
history.stateStorage.getItem('foo');
// But the values in previous entries is not updated.
history.back();
// Returns serializedUiState
history.stateStorage.getItem('foo');

// The only thing that's a little tricky is that updating a previous entry won't change future entries, so applications need to remember that.
history.stateStorage.setItem('foo', differentSerializedUiState);
history.forward();
// Returns newSerializedUiState, not differentSerializedUiState
history.stateStorage.getItem('foo');

// Entries can be removed.
history.stateStorage.removeItem('foo');
// Returns undefined
history.stateStorage.getItem('foo');

// And to fully conform to the Storage interface, the whole storage can be cleared. But this is probably rarely desirable.
history.stateStorage.clear();
```

The JS API for this `StateStorage` is identical to the existing [`Storage`](https://developer.mozilla.org/en-US/docs/Web/API/Storage) spec, the extra implicit API is in how the storage gets copied whenever the browser creates a new history entry, in response to calls to `pushState` or link clicks.

## Minor implementation note
In order to be performant, browsers should likely use [copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write) semantics for storing the actual items in the storage. Otherwise, it could be very expensive to create identical copies of large serialized items.

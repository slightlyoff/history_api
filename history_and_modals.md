# Web History and Modals

Modals are UI components that are layered on top of all other content and take
interaction focus. Some examples are:

- a `<dialog>` element, especially the [`showModal()` API](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement/showModal);
- a sidebar menu;
- a lightbox;
- a custom date picker dropdown.

There are some common implementation details that Web apps often apply to such
components:

- Trapping focus inside the modal;
- Intercepting mouse events outside the modal to disallow accidental clicks,
  and/or to close the popup;
- Sometimes, a popup may even prompt when a user attempts to close it, to ensure
  that all user data is saved;
- Intercepting <kbd>Esc</kbd> to close the popup;
- Intercepting the history Back button to close the popup, especially on
  Android.

The last point is most relevant to the Web History API: a popup may push a
non-navigational state into the history stack and close itself when this state
is popped.

As an extreme example, you can visualize a user filling in a 20 field form
with the last item being a date picker. A user may want to click the Back button
hoping to close the date picker, but instead the back button cancels the whole form because
the date picker does not integrate with the Back button.

However, the History API makes this a rather hard problem. Among the issues:

1. This state is non-navigational. URL doesn't change. When the page is reloaded,
   it's very unlikely that an app will try to open the same popup automatically.
2. It's very hard to determine when the "popup" state is popped out of history
   because both `forward` and `back` emit the same `popstate` event.
3. The `history.state` API is rather fragile: fragment navigation and nested
   `pushState/replaceState` overwrite it.
4. The history events are not cancelable. Implementing "Are you sure you want to
   leave without saving" involves buffering all history changes and re-pushing
   them back.
5. A shared component that would attempt to use the History API in this way
   can easily corrupt a web app's router.

## Solving history and modals problem

One way to solve this problem is to challenge why the History API should be
used for modals support at all. The History API integration for a modal is,
semantically, more similar to <kbd>Esc</kbd> handling than navigation.

A possible solutions, for instance, could require a popup to be built on the
`<dialog>` element. The browser would provide the Back button integration,
just as it provides <kbd>Esc</kbd> support today. The dialog API would also
need to support "save data" prompt.

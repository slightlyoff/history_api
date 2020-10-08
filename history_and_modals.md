# Web History and Modals

Modals are UI components that are layered on top of all other content and take
interaction focus. Some examples are:

- a `<dialog>` element, especially the [`showModal()` API](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement/showModal);
- a sidebar menu;
- a lightbox;
- a custom date picker dropdown;
- a fullscreen mode.

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

The History API makes handling back clicks for modals rather hard problem. Among the issues:

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

## Precedents in the Web Platform

There are many precedents of a modal-like functionality on the Web Platform today:

- The `dialog`'s `showModal()` mode cancels the dialog on the <kbd>Esc</kbd> key.
- The `requestFullscreen` mode cancels the full screen on the <kbd>Esc</kbd> key
  on desktop and on the back button on mobile.
- The dropdown of a `input[type=date]` is canceled on the <kbd>Esc</kbd> key
  on desktop and on the back button on mobile.

These features tell us that (a) there's a clear precedent for using the back
button for cancelling, and (b) this support is platform sensitive: it's expected
on a mobile platform (especially Android), but not on desktop.

## Solving history and modals problem

One way to solve this problem is to challenge why the History API should be
used for modals support at all. The History API integration for a modal is,
semantically, more similar to <kbd>Esc</kbd> handling than navigation.

Some of the possible solutions could include:

1. An extended `<dialog>` API or a new platform primitive.
2. A back button event propagated via the <kbd>Esc</kbd> key in some cases.
3. A new back button event.

### An extended `<dialog>` API or a new platform primitive

This solution would require a popup to be built on the `<dialog>` element. The
browser would provide the Back button integration, just as it provides
<kbd>Esc</kbd> support today.

There are some things that dialog API would need to improve:
1. Support more styling options to match different use cases, such as an input
   dropdown.
2. Support the "save data" prompt.

Put it together, the API use could look like this:

```javascript
const dialog = document.createElement('dialog');
document.body.appendChild(dialog);

dialog.onclose = () => {
  if (!allDataSaved()) {
    return 'Are you sure you want to close?';
  }
}
dialog.showModal();
```

The `onclose` handler is constructed similar to `unbeforeunload` event to ensure
that a "save data" prompt cannot be abused as an escape hatch to block the back
button.

If Dialog API cannot be extended in this way, there could be a similar new
HTML element that can support all of these features and allow flexible styling.
The `dialog` element itself could extend from this new type of the element.

### A back button event propagated via the Esc key

The browser could automatically propagate the back button activation as an <kbd>Esc</kbd>
key event. The main benefit of this would be that many existing popup/dialog/dropdown
implementations that already support the <kbd>Esc</kbd> key, could automatically
gain the back button support without any changes or only with minor updates.

The use of such API could then be as simple as keyboard event handling:

```javascript
const popup = document.createElement('div');
document.onkeydown = (e) => {
  if (e.code === 'Escape') {
    if (!allDataSaved()) {
      return 'Are you sure you want to close?';
    }
    popup.remove();
  }
};
```

Similarly, this event handler looks like `unbeforeunload` to ensure that this
feature cannot be abused as an escape hatch to block the back button. Consequently,
the `event.preventDefault()` can cancel the "keydown" event, but it has no
effect on the back button behavior.

### A new back button event

This approach is similar on the <kbd>Esc</kbd> event, but instead adds a new
event type. This type could either be directly tied to the back button
(`onbackbutton`) or still combine <kbd>Esc</kbd> and back button together
(`oncancel`).

The usage would be:

```javascript
const popup = document.createElement('div');
popup.oncancel = (e) => {
  if (!allDataSaved()) {
    return 'Are you sure you want to close?';
  }
  popup.remove();
};
```

A new event might be better to avoid overloading the existing event semantics
for <kbd>Esc</kbd>, but it also means that the existing components will need
to be changed to use it.

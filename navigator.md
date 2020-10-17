# Frame navigation

```
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
  getStack(): []Frame {}

  // Navigates back one Frame.
  back(): Promise<Frame> {}

  // Navigates forward one Frame.
  forward(): Promise<Frame> {}

  // Identifies any frame change that takes place.
  //   oldFrame info
  //   newFrame info
  //   replace (if applicable)
  //   action The action that caused this to change, for instance if the frame is changing due
  //       to browser actions, like a back or forward button.
  addListener('frameChange') {}
}

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

navigator.update(({url}) => {
  url.searchParams.set('foo', 'bar');
  return {url, replace: true};
});

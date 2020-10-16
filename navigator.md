# Screen navigation

```
class Navigator {
  navigate({
    name: string,
    url: URL|string,
    // Parameters that are passed as data to the Screen.
    params: {string: Any}|undefined = undefined,
    // Whether to replace the current Screen or merely push it onto the stack.
    replace: boolean = false,
    // A source Object that contains arbitrary user specified data. This is useful
    // for doing advanced history handling, like implementing a router, where data
    // needs to be passed alongside a navigation for consumption but does not need
    // to be serialized as part of the history entry.
    source: Object|undefined = undefined,
  }): Promise<Screen> {}

  // Navigate to the Screen. Ideally the name is unique in the stack. If not, it finds the first one that matches.
  navigateExisting(name: String, source: Object|undefined = undefined): Promise<Screen> {}

  // Returns the current Screen.
  getCurrentScreen(): Screen {}

  // Returns all Screens.
  getScreens(): []Screen

  // Clears all Screens.
  clear() {}

  // Identifies a navigate. This includes information such as:
  //   newScreen info
  //   oldScreen info
  //   replace (if navigate event)
  //   source
  // This event also allows preventing default to prevent the navigation from being performed or ot allow modifying
  addListener('navigate'|'navigateExisting') {}
}

class Screen {
  // Name of this Screen.
  readonly name: string;
  // URL of this Screen.
  readonly url: URL;
  // Params passed into the original navigate call.
  readonly params: {string: Any}|undefined;

  // focus - the screen navigated in.
  // blur - the screen navigated out
  // beforeRemove - the screen is about to be navigated out
  // unreachable - the screen is no longer navigable
  addListener('focus'|'blur'|'beforeRemove'|'unreachable') {}
}
```

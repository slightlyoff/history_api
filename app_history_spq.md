# App History: Security and Privacy Questionnaire Answers

The following are the answers to the W3C TAG's [security and privacy self-review questionnaire](https://w3ctag.github.io/security-questionnaire/).

**What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?**

We can think of two pieces of relevant information that might be of interest:

- The exposure, through the app history API, of information that was previously stored in the app history API, through the `appHistoryEntry.state` property. This is necessary to allow associating application and UI state with app history entries. Note that app history entries are scoped to same-origin contiguous documents, so this is not a new capability.

- The exposure of information about navigations, through the `navigate` event. This is necessary to fulfill some of the core [goals](/./app_history.md#goals) of the API, around intercepting navigations to centralize application logic and implement single-page navigation patterns. The information exposed here is the same as can be currently gathered through less-ergonomic means, e.g. global `click` handlers or a `beforeunload` handler.

**Do features in your specification expose the minimum amount of information necessary to enable their intended uses?**

Yes.

**How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?**

They do not consume such information.

**How do the features in your specification deal with sensitive information?**

They do not consume such information.

**Do the features in your specification introduce new state for an origin that persists across browsing sessions?**

No. The persistent state in `appHistoryEntry.state` is specifically scoped to session history entries, which are scoped to browsing sessions.

**Do the features in your specification expose information about the underlying platform to origins?**

No.

**Do features in this specification allow an origin access to sensors on a user’s device?**

No.

**What data do the features in this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.**

This question seems to be about cross-origin information exposure, which does not happen with this feature.

**Do features in this specification enable new script execution/loading mechanisms?**

No.

**Does this specification allow an origin access to other devices?**

No.

**Do features in this specification allow an origin some measure of control over a user agent’s native UI?**

Importantly, no. Navigations through native UI such as the URL bar or back button do not trigger a cancelable `navigate` event. See also the section ["Impact on the back button and user agent UI"](./app_history.md#impact-on-the-back-button-and-user-agent-ui) for more discussion.

**What temporary identifiers do the features in this specification create or expose to the web?**

Each app history entry has an associated auto-generated `key` property, which is a UUID. We believe this is not problematic; see more discussion in [the main explainer](./app_history.md##security-and-privacy-considerations).

**How does this specification distinguish between behavior in first-party and third-party contexts?**

It does not really distinguish. The specification's by-design segregation by browsing context, browsing session, and origin means that each party (in the rough sense) will get its own app history list, each of which behave uniformly and only give information about that party.

**How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?**

Probably no differently. If there are any differences, they will come from existing differences in how browsers model session history in such modes, but we're not aware of any at this time.

**Does this specification have both "Security Considerations" and "Privacy Considerations" sections?**

Not yet, as we're not at the specification stage.

**Do features in your specification enable downgrading default security characteristics?**

No.

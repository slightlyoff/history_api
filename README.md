# Explainers

## Introduction

Your explainer is a living document that describes the current state of your proposed web platform feature, or collection of features.

In the early phases of design, this may be as simple as a collection of goals and a sketch of one possible solution.

As your work progresses, the explainer can help facilitate multi-stakeholder discussion and consensus-building by making clear:

- the user-facing problem which needs to be solved;
- the proposed approach to solving the problem;
- the way the proposed solution may be used in practice to address the intended use cases, via example code;
- any other venues (such as mailing list, pull requests or issue threads external to the location of the explainer) where the reader may catch up on discussions regarding the proposed feature or features;
- the alternatives which have already been considered and why they were not chosen;
- accessibility, security and privacy implications which have been considered as part of the design process.

Once there is a reasonable amount of consensus on the approach and high-level design,
the explainer can be used to guide spec writing,
by serving as a high-level overview of the feature to be specified and the user need it serves.

Once the spec is written and the feature is shipped,
the explainer can then provide a basis for author-facing documentation of the new feature.

## Examples of good explainers

- [Service Workers](https://github.com/w3c/ServiceWorker/blob/master/explainer.md)
- [`paymentRequest`](https://github.com/zkoch/paymentrequest/blob/gh-pages/docs/explainer.md)
- [Web Share](https://github.com/WICG/web-share/blob/master/docs/explainer.md)
- [Viewport API](https://github.com/WICG/ViewportAPI/blob/gh-pages/README.md)
- [`EventListenerOptions`](https://github.com/WICG/EventListenerOptions/blob/gh-pages/explainer.md)
- [Intersection Observer](https://github.com/w3c/IntersectionObserver/blob/master/explainer.md)

## Tips for effective explainers:

Since your explainer may be referred to by a range of stakeholders,
not all of whom are likely to be highly motivated to spend a lot of time on it,
you should always try to keep your explainer as brief and easy to read as possible.

- Be clear about the **end-user** need, first and foremost.
- Keep it as brief and "skimmable" as you possibly can.
  - Writing succinctly is harder than writing at length. You might need to write a first draft, and then make one or more editing passes to cut down word count. This is a time investment, but will save time and energy for your readers.
  - Use bulleted lists where possible.
  - Draw attention to key points using **bold**. 
  - Keep your paragraphs and sentences short. Paragraphs should contain one idea only, and likely shouldn't be more than a couple of sentences. Break up large paragraphs as much as possible.
- Keep the language as simple as possible.
  - Not all readers will always be fluent English speakers.
  - Even if they are, they may be reading your explainer while doing three other things, with a headache and a looming deadline.
  - Be kind to your readers, since you probably want them to be kind to you.
- Wherever possible, use code examples rather than prose to "show" rather than "tell" your design.
- If you can and if it serves the document, be generous with diagrams.
  - A picture is, for most readers, much easier to process than a slab of text.
  - Always provide text alternatives for readers who may not be able to see images.
    - Simpler images may be described via an [image alt](https://webaim.org/techniques/alttext/#complex).
    - More complex images may require a longer description in the form of a footnote or appendix to the document, linked immediately after the image, with a back-link to return to the section containing the image.
- As your design evolves, keep track of and make a note of alternatives which have been considered, and your reasons for not choosing them.
  - You undoubtedly had reasons not to choose those alternatives, but reviewers and other stakeholders may not have that context. Avoid redundant "what about [already-ruled out alternative]" type questions by explaining why those alternatives were ruled out.
  - Direct readers to the appropriate participation forums, issue tracker, etc.

## Explainer Template:

# [Title]

## Authors:

- [Author 1]
- [Author 2]
- [etc.]

## Participate
- [Issue tracker]
- [Discussion forum]

## Table of Contents [if the explainer is longer than one printed page]

[You can generate a Table of Contents for markdown documents using a tool like [doctoc](https://github.com/thlorenz/doctoc).]

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

[The "executive summary" or "abstract".
Explain in a few sentences what the goals of the project are,
and a brief overview of how the solution works.
This should be no more than 1-2 paragraphs.]

## Goals [or Motivating Use Cases, or Scenarios]

[What is the **end-user need** which this project aims to address?]

## Non-goals

[If there are "adjacent" goals which may appear to be in scope but aren't,
enumerate them here. This section may be fleshed out as your design progresses and you encounter necessary technical and other trade-offs.]

## [API 1]

[For each related element of the proposed solution - be it an additional JS method, a new object, a new element, a new concept etc., create a section which briefly describes it.]

```js
// Provide example code - not IDL - demonstrating the design of the feature.

// If this API can be used on its own to address a user need,
// link it back to one of the scenarios in the goals section.

// If you need to show how to get the feature set up
// (initialized, or using permissions, etc.), include that too.
```

[Where necessary, provide links to longer explanations of the relevant pre-existing concepts and API.
If there is no suitable external documentation, you might like to provide supplementary information as an appendix in this document, and provide an internal link where appropriate.]

[If this is already specced, link to the relevant section of the spec.]

[If spec work is in progress, link to the PR or draft of the spec.]

## [API 2]

[etc.]

## Key scenarios

[If there are a suite of interacting APIs, show how they work together to solve the key scenarios described.]

### Scenario 1

[Description of the end-user scenario]

```js
// Sample code demonstrating how to use these APIs to address that scenario.
```

### Scenario 2

[etc.]

## Detailed design discussion

### [Tricky design choice #1]

[Talk through the tradeoffs in coming to the specific design point you want to make.]

```js
// Illustrated with example code.
```

[This may be an open question,
in which case you should link to any active discussion threads.]

### [Tricky design choice 2]

[etc.]

## Considered alternatives

[This should include as many alternatives as you can,
from high level architectural decisions down to alternative naming choices.]

### [Alternative 1]

[Describe an alternative which was considered,
and why you decided against it.]

### [Alternative 2]

[etc.]

## Stakeholder Feedback / Opposition

[Implementors and other stakeholders may already have publicly stated positions on this work. If you can, list them here with links to evidence as appropriate.]

- [Implementor A] : Positive
- [Stakeholder B] : No signals
- [Implementor C] : Negative

[If appropriate, explain the reasons given by other implementors for their concerns.]

## References & acknowledgements

[Your design will change and be informed by many people; acknowledge them in an ongoing way! It helps build community and, as we only get by through the contributions of many, is only fair.]

[Unless you have a specific reason not to, these should be in alphabetical order.]

Many thanks for valuable feedback and advice from:

- [Person 1]
- [Person 2]
- [etc.]

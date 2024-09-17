# Companion windows (explainer)

A window may have _companion windows_ which are similar to popus, but render within the a tab (or other top-level window), and are closed if it is closed or navigates to a different origin.

Companion windows may be:
* docked to the edge of the window (e.g., a "mole" on the bottom of the content area)
* floating above content
* attached to an element in the document (see below for more detail)

A companion window's history does not participate in the main browsing history (i.e., clicking the browser back button does not navigate the companion window).

A companion window may not, itself, have companion windows.

When a navigation within the origin occurs, the companion window remains. If the document no longer makes it attached, it will change to another presentation style (i.e., floating or docked). The new page can manipulate it, as can the user (e.g., to close it).

```javascript
// A companion window can be opened in one line.
companionWindows.open('/player.html', {name: 'player'}).then(playerWindow => {
  // ...
});

// Existing windows can be contacted and communicated with.
companionWindows.namedItem('player')?.contentWindow?.postMessage(
    {nowReadingAbout: 'hockey'}, 'https://news.example');
```

## Participate
- https://github.com/explainers-by-googlers/companion-windows/issues

## Table of Contents [if the explainer is longer than one printed page]

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Goals](#goals)
- [Non-goals](#non-goals)
- [Use cases](#use-cases)
  - [Live news video](#live-news-video)
  - [Customer support chat](#customer-support-chat)
- [Companion windows](#companion-windows)
  - [Presentation styles](#presentation-styles)
    - [CSS-oriented presentation styles](#css-oriented-presentation-styles)
    - [Anchored vs inline](#anchored-vs-inline)
  - [Application to use cases](#application-to-use-cases)
- [Detailed design discussion](#detailed-design-discussion)
  - [Lifetime](#lifetime)
  - [Partitioning & Embedding](#partitioning--embedding)
- [Considered alternatives](#considered-alternatives)
  - [Frames](#frames)
  - [Overlay window](#overlay-window)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Goals

Navigation from page to page is a key part of sites and applications on the web, a fact which remains true even as single-page apps (SPAs) increase in number. Nonetheless, different pages don't always constitute totally separate experiences &mdash; the user might be engaging with a live media stream, support session, or other adjacent experience on the site as they use it.

Pop-up windows are the classic solution for breaking out these sorts of experiences, but they have well-known drawbacks. They were open to abuse and user confusion, necessitating the advent of pop-up blockers in modern browsers. They detach fully from the original tab/window (depending on the browser) and lose their clear association with the window whence they came. (In fact, the ability to confusingly conceal pop-ups led to pop-unders and similar hostile patterns.)

This calls for a solution which:
* allows the necessary script and DOM to span pages within a site
* keeps the activity, such as media playback, going as seamlessly as possible
* fluidly changes to suit the context
* builds on existing user and developer expectations about security and privacy, making it clear how these contexts relate to one another
* behaves consistently with modern network and storage partitioning
* does not raise barriers to universal access (e.g., is consistent with best practices for accessibility and localization)
* is lightweight enough to be easy to incrementally adopt before support in browsers is widespread

## Non-goals

The solution need not displace (but should rather complement as appropriate) existing windowing tools (pop-ups, picture-in-picture, frames), navigation APIs (service workers, preloading, view transitions), and media APIs (media session, media source).

The solution need not obsolete single-page apps. It will remain useful for some sites to be structured as a single page &mdash; just not as necessary for some use cases.

The solution need not support every imaginable use of long-lived context or state. It's enough to solve some use cases well, and leave others for other APIs, like workers and storage.

## Use cases

[Describe in detail what problems end-users are facing, which this project is trying to solve. A
common mistake in this section is to take a web developer's or server operator's perspective, which
makes reviewers worry that the proposal will violate [RFC 8890, The Internet is for End
Users](https://www.rfc-editor.org/rfc/rfc8890).]

### Live news video

Informed Irene is reading about current events on the website of National News. The website includes a live player showing coverage of a breaking news event, which Irene is paying some attention to while she reads the headlines. When she clicks on an article about a major scientific discovery, though, the live news video stops as the browser navigates to the next page, disrupting Irene's ability to follow the news coverage.

Irene would have preferred it if the stream could have continued playing as Irene started reading the article.

### Customer support chat

Customer Casey is trying to obtain warranty service for a product he ordered from Widgets, Inc. He opens the support feature and, after waiting twenty minutes, is connected to a support agent. When the agent asks him for his order number, he clicks on the user profile page to locate his order. This navigation unloads the customer support session, disconnecting him from the agent.

Casey would have preferred it if the support session remained usable as he navigated the site.

## Companion windows

Each top-level traversable has a collection of _companion windows_, which are similar to auxiliary (pop-up) windows in that they are not confined by the lifetime of a single document, but similar to frames in that they form part of the user's experience browsing a single site (within or immediately adjacent to the content area).

The core of this API resembles both `window.open` and `HTMLIFrameElement`.

```webidl
[SecureContext, Exposed=Window]
interface CompanionWindow {
  attribute DOMString name;
  void close();

  readonly attribute WindowProxy? contentWindow;
};

[SecureContext, Exposed=Window]
interface CompanionWindows {
  iterable<CompanionWindow>;
  readonly attribute unsigned long length;
  getter CompanionWindow item(unsigned long index);
  CompanionWindow namedItem(DOMString name);

  Promise<CompanionWindow> open(USVString url, CompanionWindowOpenOptions options = {});
};

partial interface Window {
  [SameObject, SecureContext] readonly attribute CompanionWindows companionWindows;
};

dictionary CompanionWindowOpenOptions {
  DOMString name;
};
```

### Presentation styles

A companion window can be presented in several different ways, controlled by the top-level document. If left unstyled, the user agent ensures it is presented in a suitable way.

* **docked** to an edge of the content area (most likely, the bottom), with lightweight decorations to allow closing and possibly other operations
* **floating** within the content area, likely with an affordance for moving it around
* **anchored** to the document (but presented on top of it), e.g. using CSS anchor positioning
* **inline** within the document, by using a new element which doesn't constrain the lifetime of the companion window but does define where it appears and accepts input (allowing document content to potentially overlay the companion window)

It is unlikely that both _anchored_ and _inline_ are necessary, so one or the other should be chosen.

The presentation style may influence how an author-supplied size is used, and whether the window is user-resizable. In any case, `width` and `height` properties are likely.

#### CSS-oriented presentation styles

Instead of selecting a presentation style (and potentially other customization) via JavaScript, this could be done using CSS. This allows using several existing facilities in CSS for things like media queries and feature detection, but does require some quirks to account for the fact that the stylesheet, being part of the top-level document, will be unloaded &mdash; so the user agent will have to persist some style after that to ensure continuity.

This would likely involve exposing a `CSSStyleDeclaration` attribute on the `CompanionWindow` and defining a `::companion-window(name)` pseudo-element to allow selecting these windows declaratively, since they don't strictly speaking form part of the document (and thus wouldn't be reachable with a selector).

```css
::companion-window(player) {
  companion-window-presentation: inline, docked bottom;
}
```

If anchoring is used, this is also where the styles that are required to do anchoring could go.

#### JS-oriented presentation styles

On the other hand, this could be managed by script (which has the advantage that it's clearer how its state is mutated and maintained separately from the main document).

```js
let cw = await companionWindows.open('/player.html', {position: {anchored: true, docked: 'bottom'}});
cw.reposition({floating: 'bottom right'});
```

It's also not as natural to express an order of these: a sequence of positions would work but is cumbersome, and relying on property order isn't typical in WebIDL.

#### Anchored vs inline

Anchoring above the document creates more consistency with pop-up windows, in that the companion window would never be interleaved with content in the page. It is likely to be simpler to implement, since the docked and floating presentation styles also require rendering on top, and changing a document between being embedded in a frame-like way and not may not be easy in all existing browser engines.

Inline rendering, by projecting the window contents into a `<cwframe>` element, is more in line with how authors usually embed content of various kinds when they want it to appear as part of their page layout. It also allows content, such as tooltips and cookie consent prompts, to appear on top of the companion window (rather than having a surprising stacking order), and provides a natural place for the window to appear in reading order as well (which is harder to automatically infer from CSS anchoring, which is quite general). There is some spec complexity associated with adding an element.

### Application to use cases

National News embeds their player in a companion window (which is initially inline or anchored, but may become floating or docked as Irene scrolls). When Irene clicks to the article, the player window continues to run. If the article page styles it appropriately, it may become embedded in that page. Otherwise, it becomes docked to the edge of the content area. Either way, Irene can continue to monitor the breaking news event as she reads.

Widgets, Inc. launches their support session as a docked companion window. As long as Casey is browsing the Widgets, Inc. site, he is able to keep the support session active and readily accessible, even as he retrieves his order number.

## Detailed design discussion

### Lifetime

One of the trickiest things is determining the lifetime and style of something which is both attached to a tab and yet beyond the life of a document. Working backward from windows (rather than making frames lives longer) turns out a little easier, because windows already have somewhat independent lifetimes.

Nonetheless this issue can't be entirely dodged. For example, if a companion window is styled in a particular way but is not styled by the next document, it needs some kind of hysteresis to reduce jarring changes in its presentation on navigation, and also to handle the fact that if it _is_ styled by the next document, that style or script may take some time to load. This will probably end up with an `auto` value which has some bespoke behavior.

However, the lifetime of a companion window shouldn't be totally unbound by its opener, since the address bar and other affordances lead the user to believe it's acting as part of the top-level web site, and in fact that reflects how its storage will be partitioned. Thus it is necessary and appropriate for its lifetime to end once the top-level origin changes.

### Partitioning & Embedding

Companion windows sits somewhere between iframes (embedded) and popup windows (not embedded), which raises some questions about how they should be treated by features that care about ancestry.

It seems natural in most cases related to security and privacy to treat it as embedded, and existing solutions should translate fairly well (since they are linked to an origin that is similar to an iframe's parent or top origin).

A companion window's storage and network resources are partitioned according to the opener's top origin/site (or equivalent in the browser's partitioning scheme).

Similarly, a companion window respects `X-Frame-Options` and CSP's `frame-ancestors` directive. More specific versions might be added in the future, but this seems most likely to respect the intent of existing usage.

## Considered alternatives

### Frames

Instead of managing these as separate windows (with separate browser or platform UI), a new kind of frame (most similar to fenced frames) could be used.

This gives a great deal of familiarity and flexibility, but doesn't naturally answer the question of what happens when a navigation leads to a page which doesn't, or doesn't immediately, provide a place for the previous content to go.

It also means that subsequent pages the user visits _must_ be aware of the possible existence of the frame. By contrast, the companion windows approach allows the browser to manage the presentation of the windows in that case, without modifying other pages in many cases. The browser is also reasonably well positioned to manage the presentation in a platform-appropriate way.

### Overlay window

A transparent "overlay window" over the document which stays the same between origins could work. Scripts could then use something similar to worklets' `addModule` to load script into it. It would lend a great deal of flexibility in how any persistent elements are presented and behave, and wouldn't require browsers to cover the desired use patterns; instead, authors could keep up with design trends.

On the other hand, this is likely to be somewhat confusing, may cause contention between pages, long-lived memory usage, persistent bad states (potentially across reloads), and other issues.

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Adithya Srinivasan
- Ari Chivukula
- Camille Lamy
- Charlie Reis
- Frank Liberato
- Ghassan Kara
- Hannah van Opstal
- Johann Hofmann
- Kevin McNee
- Liviu Tinta
- Tommy Steimel
- Vlad Levin
- Zachary Cancio

and those others we've consulted (but I haven't yet sought permission to acknowledge)
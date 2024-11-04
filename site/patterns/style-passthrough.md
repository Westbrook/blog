---
layout: layouts/post.html
title: Style Passthrough Elements
date: 2024-11-01
modified: 2024-11-01
---

One of the hardest-to-explain values (because it's often seen as a cost, instead) of working with Web Components and their various APIs is the amount of untilled soil underneath the feet of developers who both write and consume such components. Whether due to [_certain_ frameworks](https://custom-elements-everywhere.com/) choosing not to have the best support for the DOM, lateness of support of [various](https://developer.mozilla.org/en-US/docs/Web/CSS/:state) [features](https://caniuse.com/mdn-javascript_statements_import_import_attributes_type_css) in browsers, or simply unexpected [holes in the platform](https://github.com/w3c/csswg-drafts/issues/6867) (all examples of a larger situation, not a complete snapshot of reasons), for every new and exciting pattern such a developer can pluck from the tree that is the web platform, there seems to be two or ten that they can't quite get at to see how sweet its juice.

Sometimes this can prevent the rightful exploration of the field, but sometimes necessity breeds invention! No, not all invention is good, and even less of that becomes "best practice", but you won't know till you try and the web isn't going to get better and more interesting on its own. Not without invested people continuing to share about the zany ideas they have for making it better.

Recently, as I build a new component library for the company that I am helping to start, I've begun playing with a pattern for styling a Custom Element that wraps a native built-in element in a way that I think is interesting. You, too, might want to check out...

```js
const template = document.createElement("template");
template.innerHTML = /*html*/`
  <button><slot></slot></button>
`;

const sheet = new CSSStyleSheet();
sheet.replaceSync(/*css*/`
/* Defaults */
:host {
	display: contents;
	/* Style the "button" here, especially in ways that a consumer could _override_. */
}
/* Pass all styles from :host to button */
button {
  all: inherit;
  display: inline-flex;
  box-sizing: border-box;
}
@media (prefers-reduced-motion: no-preference) {
	/* Because accessibility means so much more than just Tab keys and screen readers! */
	:host {
		transition:
			outline 0.3s ease-in-out,
			outline-offset 0.3s ease-in-out;
	}
}
/* Required */
/*
 * All of our elements need _something_ to be a specific way, like accessibility.
 * This prevents the consumer from futzing with those things.
 **/
button:not(:focus-visible) {
	outline: 2px solid transparent;
	outline-offset: 0px;
}
button:focus-visible {
	outline: 2px solid royalblue;
	outline-offset: 2px;
}
/* Overrides */
:host([disabled]) {
	pointer-events: none;
}
`);

class CustomButton extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: "open" });
    this.shadowRoot.appendChild(template.content.cloneNode(true));
    this.shadowRoot.adoptedStyleSheets = [sheet];
  }
}

customElements.define("custom-button", CustomButton);
```

Yes, I've both stripped this bare for the sake of the example, but included some _opinionated_ things to really clarify the pattern. Let's dig in!

## The `template`

No, we don't have [HTML Module Scripts](https://github.com/w3ctag/design-reviews/issues/334) or [DOM Parts](https://github.com/WICG/webcomponents/blob/gh-pages/proposals/DOM-Parts.md), yet, so we're working from a pretty simple template.

Yes, I've scooped out all the fancy things (read extra content, slots, etc.) that would make customizing a `button` element like this _really_ worth it.

All in all, there's not much _new_ here, so let's move on.

## The element definition

No, we are not going in code order.

No, I've not used any special library (though I often [use one](https://lit.dev/), and you might also benefit from doing so). This pattern applies regardless.

Yes, there is lots of "extra" functionality you could build into a custom element like this. Share back what you'd add via [Mastodon](https://mastodon.social/@westbrook). 🙏

## Those styles, though... 💅

No, [CSS Module Scripts](https://caniuse.com/mdn-javascript_statements_import_import_attributes_type_css) are not fully a thing, yet. So...

Yes, we've put out CSS _in_ out JS, but that's just an implementation detail.

The heavy lifting is as follows:

```css
:host {
	display: contents;
}
button {
	all: inherit;
}
```

First, we use `display: contents;` on the `:host` to allow its child (the `<button>`) to participate in layout rules applied by its ancestors; like `display: flex;` and `display: grid;`.

Then, we apply `all: inherit;` on the wrapped element so that all styles applied to the root element are passed through to it. And, then you're done!

Well, actually no.

Now you've created a shell that accepts styles from the outside without the need for a wall full of [CSS Custom Properties](/styling/css-custom-properties/) or needing to document the availability of [CSS Parts](/styling/css-parts/) within your element of any kind. Yes, the consumer of your element will need to address rules to the `custom-button` tag name, as opposed to the native built-in of `button`, but they will have full power over how that `custom-button` in a way that CSS Custom Properties cannot (at least not without A LOT of custom properties) or the need for new syntax (or [not so new](https://caniuse.com/mdn-css_selectors_part)) like `::part(button)` to do so. No, that's still not all.

Being the `button` element is within your shadow root, you still have access to styling bits and pieces of it as you'd please. Some of these could be defaults, applied via the `:host` selector so that external CSS could override it. Imagine your button is a "pill button":

```css
:host {
	display: contents;
	border-radius: calc(Infinity * 1px);
	align-items: center;
	line-height: 1.5;
	padding: 5px 10px;
	border: 2px solid;
	font-weight: 500;
	cursor: pointer;
	width: max-content;
}
```

All of these things power the default delivery of the `custom-button` element while being applied at `:host` specificity, so as not to get in the way of your consumers. However, some things might not be "for consumer customization", like maybe an accessibility feature, or multiple accessibility features:

```css
@media (prefers-reduced-motion: no-preference) {
	:host {
		transition:
			outline 0.3s ease-in-out,
			outline-offset 0.3s ease-in-out;
	}
}
button:not(:focus-visible) {
	outline: 2px solid transparent;
	outline-offset: 0px;
}
button:focus-visible {
	outline: 2px solid royalblue;
	outline-offset: 2px;
}
```

Here we see styles applied directly to the `button` selector itself in `button:not(:focus-visible)` and `button:focus-visible`, something that a consumer would not be able to target to change or, worse, to remove. _Always_ the `button` element contained in the `custom-button` will be delivered with a `2px solid royalblue` outline offset by `2px`, no matter what. We've even ensured that in _most_ cases that outline in neatly transitioned between its two states. That is unless user preferences for reduced motion prevent that rule from being applied.

The `transition` rule is applied to the `:host` for inheriting, which means that a consuming developer could alter those styles. Removing them won't be the worst thing in that they are a progressive enhancement when user preferences allow their presence. Changing them to apply 100% of the time would be less than optimal, and I wonder if there are enhancements to this technique that could allow that not to be so. Do you see any? I'd love to hear them.

## You shall...pass?

Yes, you shall. And, you little styles, too!

Even as I prepared this post, I found little tweaks and edits to help me think more deeply about this pattern. Previously, I thought it might require [CSS @layer](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer) (which is super cool), but it didn't. Above, I noted an issue around consumers changing the application of `transition` rules, which hopefully you or your intrepid reader friends can point a way out of; `!important` much? And, I realized that without the handful of attributes changed based on "states" and "variants" that I leverage this pattern with, it's less clear why surfacing the element for styling this way might be beneficial:

```css
/* Sizes */
:host,
:host([size="m"]:where(:not([icon-only]))) {
	gap: var(--rv-rhythm);
	padding: 0 calc(1.5 * var(--rv-rhythm) - var(--rv-border-m));
}
:host([size="s"]) {
	font-size: calc(13rem / (16 * var(--rv-font-scale)));
	gap: calc(0.5 * var(--rv-rhythm));
	padding: 0 calc(1 * var(--rv-rhythm) - var(--rv-border-m));
	line-height: calc(3 * var(--rv-rhythm) - (2 * var(--rv-border-m)));
}
:host([size="s"][icon-only]) {
	padding: 0 calc(0.5 * var(--rv-rhythm) - var(--rv-border-m));
	height: calc(3 * var(--rv-rhythm));
}
/* Variants */
:host([variant="primary"]) {
	color: var(--content-on-brand);
	--pill-button-background: var(--background-brand);
}
:host,
:host([variant="secondary"]) {
	--pill-button-background: var(--background-tertiary);
}
:host([variant="secondary"][quiet]) {
	color: var(--content-secondary);
}
:host([quiet]) {
	color: var(--content-primary);
	--pill-button-background: transparent;
}
:host([selected]) {
	--pill-button-background: var(--background-brand-secondary);
	border-color: var(--border-brand);
	color: var(--content-brand) !important;
}
/* Interactions */
:host(:hover) {
	color: var(--content-primary);
	background-image: linear-gradient(to right, var(--background-secondary), var(--background-secondary)),
		linear-gradient(to right, var(--pill-button-background), var(--pill-button-background));
}
:host([variant="primary"]:hover) {
	color: var(--content-on-brand);
}
:host([quiet]:hover) {
	color: var(--content-primary);
}
```

But, maybe you find it beneficial, too, and together we can find even more productive use cases to which we can apply this pattern. Let me know about them on [Mastodon](https://mastodon.social/@westbrook). 👋

Is that all, now? Yes.

Try it out in [Codepen](https://codepen.io/Westbrook/pen/poMKyxB).
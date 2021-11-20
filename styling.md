 Sometimes the best way to learn something is to copy the way that it had been done before enough times that the work involved actually sneaks into your consciousness through osmosis and one day you find yourself actually to be a specialist at said task. I've taken part in this time-honored tradition countless times over the course of my career. Starting quite early on when I copy and pasted every single `<link rel="stylesheet" href="/style.css" />` into every single HTML page I wrote. Even just today I took a peek on MDN to make sure I hadn't messed that up. Lately, I've been using tools like `plop` and `npm init @open-wc` to copy and paste entire file systems into a repo. There may be a useful investigation into the ratio of copy and pasting I do to the amount of work experience I've accumulated to be taken on in the future... however, we've gathered here today to learn about styling `LitElement`s and what I was starting to get at is how we should probably copy something that works a little like a `LitElement` that we can style today in the browser and use it as a way to level up our approach to both leveraging and creating an API for styling custom elements.

Luckily, all custom elements work a little like native elements. Very few native elements leverage _all_ of the APIs that we have access to in user space, but one that does a pretty good job of this is the `details` element. 

```html
<details>
  <summary>Details</summary>
  Something small enough to escape casual notice.
</details>
```

It's not immediately clear from just looking at it, but if you inspect [this demo](https://webcomponents.dev/edit/eQFFNYrcjIYdqm2LXyqA) (with "Show user agent shadow DOM" toggled on in your Chrome DevTools) a better picture of the various APIs at play here will become available. First, as you toggle the details open and closed by clicking on the `<summary>` elements, you'll see the `open` attribute hung off of the `<details>` element maintaining the state of the visibility of the extended content. The majority of the content of the element is also applied as light DOM so that it can be styled from the outside. This light DOM will serve to allow custom elements to present content while `:not(:defined)`; before they've been upgraded. Inspect a little deeper and the use of multiple slots to project the `<summary>` and "content" into different parts of the shadow DOM. Dive into the `<summary>` element and you'll see decorative DOM structured at pseudo-elements so that they can be addressed with styles, all the while giving it a sensible default from which to start. Just about the only thing that it _doesn't_ do is apply default styling on slotted content, which can be something to learn from in and of itself. So, let's get into copying!

Of course, to style a `LitElement` to begin with you need to know how to build one. We'll take a quick look at how our `<custom-details>` element will come together next, but if you want to make sure you've been fully primed on creating a `LitElement`, check out Benny Powers' great post ["Let's Build Web Components! Part 5: LitElement(https://dev.to/bennypowers/lets-build-web-components-part-5-litelement-906). And, if that's not enough for you, flex your coding muscles over these [Open Web Components code labs](https://open-wc.org/guides/developing-components/codelabs/#lit-html--lit-element-basics).

## Let's buid a <custom-details> element!

Did you get a good enough intro to the `<details>` element above? Well, I hope so, because you're officially running QA on our brand new [`<custom-details>` implementation](https://webcomponents.dev/edit/L0gbywmSINP3KweL4Otc):

```javascript
import { LitElement, html, property, css, query } from 'lit-element';

export class CustomDetails extends LitElement {
  static styles = [css`
    :host {
      display: block;
    }
    
    .summary ::slotted(:first-child):before {
      content: '►';
      vertical-align: middle;
      font-size: 0.75em;
      margin-inline-end: 0.4rem;
      font-family: sans-serif;
      overflow: hidden;
    }

    :host([open]) .summary ::slotted(:first-child):before {
      content: '▼';
    }
  `]

  toggle() {
    this.open = !this.open;
  }

  private handleClick(event: Event & { target: HTMLDivElement}) {
    this.toggle();
  }

  private handleKeyup(event: KeyboardEvent) {
    if (event.code === 'Space') {
      this.toggle();
    }
  }

  private handleKeypress(event: KeyboardEvent) {
    if (event.code === 'Enter') {
      this.toggle();
    }
  }

  private handleSlotchange(
    {target}: Event & {
      target: HTMLSlotElement
    }
  ) {
    const summary = target.assignedElements()[0];
    if (summary) {
      summary.setAttribute('tabindex', '0');
    }
  }

  @property({type: Boolean, reflect: true}) open = false;

  @query('.summary') summaryElement!: HTMLDivElement;

  render() {
    return html`
      <div
        class="summary"
        @click=${this.handleClick}
        @keyup=${this.handleKeyup}
        @keypress=${this.handleKeypress}
      >
        <slot
          name="summary"
          @slotchange=${this.handleSlotchange}
        ></slot>
      </div>
      ${this.open
        ? html`<slot></slot>`
        : html``
      }
    `;
  }
}

customElements.define('custom-details', CustomDetails);
```

Yes, if you had spent the time beforehand to set up some visual regression tests, or just had a good eye, you will notice that there is a slight visual change in how the arrow is sized/positioned/shaped after the refactor to `<custom-details>`, but, for the sake of simplicity, I'm OK with leaving that alone, if you are. We could spend some time adding some extra meat to our implementation now, or we could dive into styling some of the light DOM that goes into leveraging this:

```html
<custom-details>
  <div slot="summary">Details</div>
  Something small enough to escape casual notice.
</custom-details>
```

Sure, there's not a lot of light DOM here. That will help us stay focused; and, if/when we need it, we can expand our usage example to explain appropriate realities. But first, let's do a quick review of the functionality we've built into `<custom-details>` before we get into that.

First, we defined the `<custom-details>` element, which means you can use it over and over again across our page or application, yay!
- It manages the visible state of the detail content via the `open` attribute that is bound to the property of the same name. We've surfaced a `toggle()` method that will allow for the value of `open` to be toggled easily from the outside (something the `<details>` element doesn't do natively). This isn't a huge addition, in that `details.toggleAttribute('open')` or `details.click()` would each surface the same functionality on the native element, but I wanted to make sure we're all on the same page.
- We project both `summary` (a slight departure from creating a second `<custom-sumarry>` element, but necessary as we don't yet have access to the [imparative slotting](https://github.com/WICG/webcomponents/blob/gh-pages/proposals/Imperative-Shadow-DOM-Distribution-API.md) techniques needed to fully copy the browser architecture here otherwise) and _default_ content into slots available internal of the element's shadow DOM.
- With a `click` event listener bound to the element wrapping the `<slot name="summary">` element we are able to toggle the `open` values in response to pointer interactions.
- In support of accessible interactions, we've added the same management to the `keyup` event for the "Space" key and the `keypress` event of the "Enter" key.
- To make interactions with the `[slot="summary"]` element, we giving it a `tabindex` of `0` so that it can be focused.
- The presence of the _default_ slot is managed by the value of `open` meaning that the browser will handle hiding the details "content" when the element is not `open`.
- Depending on the value of `open` the "arrow" displaying alongside the summary content is rotated.

Note: Without this imperative API mentioned above, we're not able to restrict the content in the "summary" slot to a single element, as we see with `<details>`. We could go an extra step without default styles and hide everything with something like `::slotted([slot="summary"]:not(:first-of-type)) { display: none; }`, however that seems like a little intrusive to me _and_ there's a possibility not doing that opens up new and interesting use cases that our users would never discover otherwise.

And there you have it, the `<custom-details>` element. So, let's get to styling!

# Styling light DOM from the outside

One of the best parts of matching the content API of the native `<details>` element here is that because of this we can style almost every part of the actual consumption of our `<custom-details>` copy from the light DOM as well. That means if we like the styles that [MDN uses](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/details) in its `<details>` demo, we can [leverage them](https://webcomponents.dev/edit/ZEuJxm41pjpOTBP8EyDV?file=www%2Findex.css), too!

With a slight renaming of the selectors, of course:

```css
custom-details {
    border: 1px solid #aaa;
    border-radius: 4px;
    padding: .5em .5em 0;
}

[slot="summary"] {
    font-weight: bold;
    margin: -.5em -.5em 0;
    padding: .5em;
}

custom-details[open] {
    padding: .5em;
}

custom-details[open] [slot="summary"] {
    border-bottom: 1px solid #aaa;
    margin-bottom: .5em;
}
```

Add this snippet to the CSS applied to the host in which our `<custom-details>` lives and you too can have the MDN demo styled elements in your site, today.

Yes, customizing the delivery of a `LitElement` takes no more than that small amount of copy and paste and edit. That is, assuming it is appropriately prepared to do so, of course. Using your custom element to encapsulate functionality while accepting all content from the outside means that consumers have the relatively unbridled ability to deliver your custom element how they'd like in their own application. In that way, our `<custom-details>` should be able to serve the use case of one of the most prolific consumers of `<details>` elements that I know, GitHub. If it's not something you've thought of in the past, go check out the ["Watch" UI](https://github.com/polymer/lit-html) on any GitHub repo and you'll find that it is in fact powered by the `<details>` element:

![A screenshot of the `lit-html` watch UI from GitHub in the "watched" state.](https://dev-to-uploads.s3.amazonaws.com/i/en4o0bgo8xbmqdg66c4n.png)

While you're at the `lit-html` repo, I suggest you go ahead and star it before we move on. Just to share the love, you know... When you're done, take a look at how the following CSS [makes for a close facsimile](https://webcomponents.dev/edit/qAnUUE044B4TtUhqUjzM) with our `<custom-details>` element:

```css
:root {
  --color-text-primary: rgb(201, 209, 217);
  --color-btn-text: #c9d1d9;
  --color-btn-bg: #21262d;
  --color-btn-shadow: 0 0 transparent;
  --color-border-overlay: rgb(48, 54, 61);
  --color-bg-overlay: rgb(33, 38, 45);
}

custom-details {
  position: relative;
}

[slot="summary"] {
  padding: 3px 12px;
  font-size: 12px;
  line-height: 20px;
  color: var(--color-btn-text);
  background-color: var(--color-btn-bg);
  border-color: var(--color-btn-border);
  box-shadow: var(--color-btn-shadow),var(--color-btn-inset-shadow);
  transition: .2s cubic-bezier(.3,0,.5,1);
  transition-property: color, background-color, border-color;
  position: relative;
  display: inline-block;
  font-weight: 500;
  white-space: nowrap;
  vertical-align: middle;
  cursor: pointer;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
  border: 1px solid;
  border-radius: 6px;
  -webkit-appearance: none;
  -moz-appearance: none;
  appearance: none;
}

[slot="summary"]:focus {
  border-color: var(--color-btn-focus-border);
  outline: none;
  box-shadow: var(--color-btn-focus-shadow);
}

[slot="summary"]:before {
  display: none;
}

.menu {
  position: absolute;
  top: 100%;
  left: 0;
  width: 300px;
  height: auto;
  max-height: 480px;
  margin: 8px 0 16px;
  padding: 7px 7px 7px 16px;
  font-size: 12px;
  border-color: var(--color-border-overlay);
  border-radius: 6px;
  box-shadow: var(--color-shadow-large);
  z-index: 99;
  display: flex;
  overflow: hidden;
  pointer-events: auto;
  flex-direction: column;
  background-color: var(--color-bg-overlay);
  border: 1px solid var(--color-select-menu-backdrop-border);
  color: var(--color-text-primary);
}
```

I've not gone so far as completely recreating GitHub's CSS or hunting down every single one of it's CSS Custom Properties, but hopefully, it's becoming clear how you _could_ do such a thing and delivery the GitHub watch UI in your application with our `<custom-details>` component:

![A partial reimplementation of the GitHub watch UI built with our `<custom-details>` element.](https://dev-to-uploads.s3.amazonaws.com/i/8zs3mba6b0d7ahkdw5se.png)

## Reset vs Normalize

So far we've taken a hybrid technique in applying styles to `<custom-details>`. We've set a light baseline of default styles internally while relying on the idea that the majority of our content is applied via slots allowing for that content to be customized with styles from the outside. This takes a somewhat parallel path to the more established CSS technique of normalization; which is starting from a CSS base layer that ensures that all browsers will deliver content the same before applying your own customization. Often normalization is contrasted with CSS resetting, wherein you add rules that _unset_ the styles that an element has been given so that you can rebuild them in your own image. I've recently been introduced to this great [comic retelling of the two techniques by Elijah Manor](https://elijahmanor.com/blog/css-resets).

![CSS Reset vs CSS Normalize](https://dev-to-uploads.s3.amazonaws.com/i/6mxr9ej4uxlyaqosw710.jpg)

With this in mind, and in reverence to the great web holiday [CSS Naked day](https://css-naked-day.github.io/), we could take simply skip the settings of default so that we didn't need to reset anything. Here's a [naked `<custom-details>`](https://webcomponents.dev/edit/aAM9wA0KzKWdQ2Rf30nk). It's actually kind of nice. What's more, omitting 100% of the styles means that consumers can make up their own minds as to what the element should look like when delivered on a page or in an application. In fact, if they _wanted_ to make it look "exactly" like a `<details>` element, they'd need nothing more than add the following CSS from the outside:

```css
custom-details {
    display: block;
}

[slot="summary"] {
    display: flex;
}

[slot="summary"]:before {
    content: '►';
    display: inline-block;
    align-self: flex-end;
    font-size: 0.75em;
    margin-inline-end: 0.4rem;
    font-family: system;
    overflow: hidden;
}

custom-details[open] [slot="summary"]:before {
    content: '▼';
}
```

The theory here is that when starting from naked the amount of CSS required to get to "just right" _should_ be less, and in that way, someone wanting to reimplement the watch UI usage from GitHub would be able to apply the CSS outlined above to do so while omitting the styles that were unsetting previously applied defaults:

```css
[slot="summary"]:before {
  display: none;
}
```

## From the outside in

There are lots of cases where a custom element accepts all of its content from light DOM and customizing the visual delivery of that functionality from the outside might just be the perfect fit for that pattern. This style of progressive enhancement is really powerful in that it pairs well in contexts where JS is not or is not yet available by allowing those external styles to be available even before your custom element is upgraded by the browser. However, it also presents the difficulty of needing to reapply the custom styles in every DOM tree that leverages your component. When that is across successive pages, the difficulty this presents might not be all that large as it is the common reality of native elements. However, if this is across successive sub-trees (as created by shadow roots), then the need to reapply those styles again and again across the application can become much more of a nuisance. In the following section, we'll look at ways to style out components from the inside out.

### Outside in: Advanced?

Even when expecting little or no styles from the outside in, there is the possibility that consumers of your element will need to style them before the [JS that defines it](https://kryogenix.org/code/browser/everyonehasjs.html) is able to do so. I find this to be a fairly uncovered part of the custom element delivery story to date. Part of this is due to non-encapsulated styles being a fairly uncovered story in the frontend development ecosystem, to begin with, but also because the possibility to do so is a capability somewhat unique to custom elements. Traditionally, a JS framework component only exists when it is available because only when it is available can it be rendered to the DOM. Even when you move a framework solution off of the client and into the arena of server-side rendering (SSR), on the server all the JS is available, and on the client, all of the output of the JS is available (styles, DOM, etc.) even in the uncanny valley where the JS isn't quite available yet. However, all the light DOM that you decide to place in `<my-element>` is wrapped in what amounts to a `span` until `customElements.define('my-element', MyElement);` is actually run in the browser. In this way, whether we're waiting for a giant blob of all the JS at the initial load of our application/page, or we're waiting to request more nuanced chunks over the course of a user's journey there is a time fairly unique to custom elements that we can choose to address.

Jake Trent [outlines a great convention](https://jaketrent.com/post/package-json-style-attribute) in need of possible standardization when he discusses the use of the `style` attribute of a `package.json` file. The ability to surface the concept of "styles that can make the use of this component easier/nicer" seems very powerful in conjunction with the no-JS/pre-JS realities outlined above. Whether you're encapsulating all of the style delivery of your element's internals inside of your element, as we'll get deeper into next, or assuming consumers will fully customize them from the outside 100% of the time, the idea that a "baseline" or "loading" or "helper" set of styles could be defined as part of your component's package is really valuable. Along those same lines, the growing capabilities of the `exports` attribute in `package.json` files further opens avenues for investigation in that it allows for even more precise asset requests, and could theorize surfacing, not just a single "baseline" or "loading" sheet, but multiple sheets defining those things and "helpers" and "themes" and on and on.

If this sounds interesting to you, I'd very much love to hear your thoughts in the comments below. "Convention" isn't quite strong enough a word here to really power the beautiful future that I could see opening up behind the doors this sort of technique could open. I'd be very excited to partner with other invested developers to solidify this as a "standard", possibly as part of the ongoing work of the [Web Components Community Group](https://www.w3.org/community/webcomponents/) at the w3c. 

# Light DOM with all internal styling

When you put all of your styling inside of your element you leverage the encapsulation of the platform to structure a more predictably reusable component. 

## static styles = [ css&#96;&#96; ]

Working with the `css` template literal tag you are leveraging a graceful degradation series that leverages `document.adoptedStyleSheets` where available in partnership with `CSSStyleSheet` objects for performant CSS content sharing across multiple components which degrades to in DOM `<style>` elements when not available (browsers can choose to deduplicate these via content hashing, but the JS scope has no direct control over this process) before falling back to the ShadyDOM polyfill in browsers that do not support shadow DOM at all.

## importing stylesheets

Leveraging CSS modularized by the `css` template literal tag means that you can import that CSS from outside of your custom element module. One immediately apparent benefit of this might be easier inclusion in tooling pipelines, particularly the ability to write CSS files that get processed to a JS file that exports these styles. However, the more powerful possibility that this opens is the ability to share the same styling across different custom elements. This can take the shape of CSS resetting/normalization, sharing a themeing across your components, packaging utility helpers that your components can opt-into as needed, or any number of patterns you may have previously leveraged CSS pre-processing to manage. We'll talk more deeply about the possibilities below.

## &lt;style&gt; tags for local dynamism

As fewer browser and users rely on the ShadyDOM polyfilling (in fact, upcoming releases of `lit-element` are moving away from offering this code path by default) leveraging `<style>` tags directly in the shadow DOM of your element. While this shouldn't be the _default_ path to styling your element, it does open the ability to bind CSS properties to properties in your element. This can add a powerful dynamism to your elements wherein consumers can directly affect the styling of your element via its attribute/property API.

# css tag advanced

Once you start importing your tagged CSS literals, an opportunity to use some advanced patterns in this are open up. It can be easy to fall for the simplicity of one style sheet for one component. I'd actually say there's a benefit to that in small scale systems. However, as your project expands needing to share things across multiple components will become more and more important in ensuring conformity across the system. Importing CSS literals can support that in a number of ways.

## sheet composition

Include multiple whole sheets in your element:

```javascript
static styles = [
  cssNormalize,
  cssComposition,
  cssUtilities,
  cssBlock,
  cssExceptions,
  cssMyElement
];
```

## rule composition

As outlined in [Not Another To-Do App: Part 8](https://dev.to/westbrook/not-another-to-do-app-part-8-3lic) we can compose specific individual rules as well:

```javascript
export const formElementFocus = css`
  outline: none;
  background-image:
    linear-gradient(
      to top,
      #0077FF 0px,
      #0077FF 2px,
      transparent 2px,
      transparent 100%
    );
`;

export const formElementHover = css`
  background-image:
    linear-gradient(
      to top,
      currentColor 0px,
      currentColor 2px,
      transparent 2px,
      transparent 100%
    );
`;

export const styledInput = css`
input:focus {
    ${formElementFocus}
    /* Further :focus customizations for <input> elements */
}
input:hover {
    ${formElementHover}
    /* Further :hover customizations for <input> elements */
}
`;
```

## turning off shadyDOM

# Styling shadow DOM

## CSS Custom Properties

## CSS Shadow Parts

We've been going through a lot of fairly new concepts as part of the work of seeing all the different ways styling a `LitElement`, and we're not done yet. CSS Shadow Parts in another great way to surface API to access a specific piece of customization within your shadow root; in this case an entire element. Before we get into how we might be able to leverage this capability in our element, take a reading of ["Using CSS shadow parts in web components"](https://dev.to/43081j/using-css-shadow-parts-in-web-components-7h5) for a quick run down on how CSS Shadow Parts work.

Long story short, with the following default styles we achieve the visuals of a native details/summary pair:

```css
    :host {
      display: block;
    }

    .summary {
      display: flex;
    }

    .summary:before {
      content: '►';
      align-self: center;
      font-size: 0.75rem;
      margin-inline-end: 0.4rem;
      font-family: sans-serif;
      overflow: hidden;
    }

    :host([open]) .summary:before {
      content: '▼';
    }

    ::slotted([slot="summary"]) {
      pointer-events: none;
    }
```

And, with the `part="summary"` attribute added to our `class="summary"` element, like so:

```html
    <div
      class="summary"
      part="summary"
      tabindex="0"
      @click=${this.handleClick}
      @keyup=${this.handleKeyup}
      @keypress=${this.handleKeypress}
    >
      <slot
        name="summary"
      ></slot>
    </div>
```

We empower the ability to fully customize the delivery of our `<custom-details>` element from the outside.

Of specific note here is the move away from styling the slotted content from inside of our element, which has lower specificity than styles applied to that content from the outside. We had previously avoided this so that we could have shared access to the `:before` pseudo element from both contexts. 

# What have you been up to?

I work for [Adobe](https://www.adobe.com/) where I lead the [Spectrum Web Components](https://opensource.adobe.com/spectrum-web-components/) project. As part of that work, I leverage these and many more useful capabilities of `LitElement` to bring quality reusable components to the life that empower broader and deeper creativity on the web. If you've been using/plan to use/want to learn more about the many possibilities for styling/building/shipping custom elements in your work I'd love to hear about it in the comments below. 

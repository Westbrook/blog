---
layout: layouts/patterns.html
---

# Adopting your parent's style

After much adieu, [Constructible Stylesheets](https://dev.to/westbrook/why-would-anyone-use-constructible-stylesheets-anyways-19ng) are finally about to land in all modern browsers. This means so much for tools that have already adopted them in concert with fallbacks to previous techniques to support less forward-thinking browsers; both in form of performance and code simplicity. For tools that haven't adopted them yet, finally, there is a clear path to the [performance benefits](https://github.com/emotion-js/emotion/issues/2501) that come along with this new browser capability. It also means that there is a new possibility of finding new and interesting use cases only made available to us by the presence of this API, thanks to its broader availability. One of those includes the ability to adopt your parent's styles, which might work like this:

```js
import styles from './styles.css' assert { type: 'css' };

class ParentStylesAdoptingElement extends HTMLElement {
    constructor() {
        super();
        this.attachShadow({ mode: 'open'});
        this.shadowRoot.appendChild(
            Test.template()
        );
    }

    shadowRoot;

    static template;

    static {
        const template = document.createElement('template');
        template.innerHTML = /* html */`
            <slot></slot>
        `;
        this.template = () => template.content.cloneNode(true);
    }

    connectedCallback() {
        const hostStyles = this.getRootNode().adoptedStyleSheets || [];
        this.shadowRoot.adoptedStyleSheets = [
            ...hostStyles,
            styles
        ];
    }
}

customElements.define('parent-styles-adopting-element', ParentStylesAdoptingElement);
```

This example also uses [static blocks](/patterns/static-blocks), but they aren't specifically doing any new or interesting work in this case.

What's doing the interesting work in the `connectedCallback()`. It finds the root node of the host and then takes any [`adoptedStyleSheets`](https://developer.mozilla.org/en-US/docs/Web/API/Document/adoptedStyleSheets) it may have been provided and applies them to the host elements shadow root, as well.

```js
connectedCallback() {
    const hostStyles = this.getRootNode().adoptedStyleSheets || [];
    this.shadowRoot.adoptedStyleSheets = [
        ...hostStyles,
        styles
    ];
}
```

In this way, we've created a custom element with a shadow root that for all intents and purposes no longer encapsulates its shadow DOM from styles in its parent DOM tree. Could this be a solution to ["open-stylable" Shadow Roots](https://github.com/WICG/webcomponents/issues/909)?

Well, not a total solution... caveats incoming!

In a world where styles are applied to an HTML document with HTML (gasp!) the ceiling of the parent style adoption chain would likely be maintained by the first custom element on a page. That is to say `<link rel="style" href="styles.css">` doesn't get registered on the `adoptedStyleSheets` array. For shadow roots, it is more common to apply styles via `adoptedStyleSheets` as before [Declarative Shadow DOM (DSD)](https://web.dev/declarative-shadow-dom/) there was no way to attach a shadow root to an element without JS, but with the growth of DSD, this context, like the host HTML document, will "suffer" from the lack of documents applying their styles in this way.

- You could just apply ALL of your styles via `adoptedStyleSheets`. However, that wouldn't work without JS, and is inherently not declarative, which is the latest holy grail of the web development community.
- The style sheet resolution method could be altered to take into account non-constructed style sheets. In theory, you'd only have to construct them once, so the output of the work involved could be cached, saving time as the number of custom elements in hosts styles by `<link rel="stylesheet">` scaled up.
- `adoptedStyleSheets` could be updated at the spec level to accept non-constructed style sheets, which feels like a decent idea anyways. This also starts to point towards declarative adoption of style sheets even when not acquiring them from your parents.
- `<link rel="stylesheet">` based style sheets could be automatically applied to the `adoptedStyleSheets` array.

Much of the above involves changes to browser specifications, which may prove time or cost prohibitive. However, the idea of shared, or encapsulated, or componentized styles will continue to be with us for quite some time. That means that we'll be better off with more flexible patterns for consuming them rather than less. `adoptedStyleSheets` will be a major force in opening those up, especially in concert with [import assertions](https://tc39.es/proposal-import-assertions/), and the more we can talk about the possibilities of these new APIs, the better!
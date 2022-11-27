---
layout: layouts/post.html
---

# Class extension and registration

If you can come to the agreement that the custom element is the Sistine Chapel of custom elements and should always be delivered 100% as is, then you still have a couple of agreements that you can come to with the author of your custom element. Here I'll outline how you can take _ownership_ of the custom element and its styles, assuming that the original author is exporting the class definition of the custom element, already.

```js
export class CustomElement extends HTMLElement {
    //...
}
```
<dialog></dialog>

Assuming there is code similar to the above in the author's original custom element, then you're well on your way to owning your own custom element and managing its styles as you see fit.

As an aside: it's probably a good idea for the author of any custom element to surface the class definition of the element in this way. Not only does it mean that the class definition can be more readily leveraged in type-strict coding contexts: e.g. TypeScript, this is the basis for a quality code architecture pattern where the definition and the registration of a custom element live in different files. This allows consuming applications can have access to a custom element definition, without the side effect of registering that custom element being played immediately opening the door to more flexible lazy loading techniques.

```js
// CustomElement.js
export class CustomElement extends HTMLElement {
    //...
}
```
<dialog></dialog>

```js
// custom-element.js
import { CustomElement } from './CustomElement.js';

customElements.define('custom-element', CustomElement);
```
<dialog></dialog>

The above split also means that when taking ownership of the custom element you are not immediately required to give that element a new tag name. This means that you have the option to persist the mental model with which you were developing your application before deciding to customize the element in question. There are cases where you may want to keep the original custom element along with the one with your style customization, which would lead you to register a new tag name.

In the case that the original author has not exported the class definition, you can work around this by leveraging `customElements.get('custom-element')`, which would return you the class definition of `custom-element`, assuming that it was registered on that page. This is a quality release valve, but needing to rely on runtime code to take ownership of a custom element's styles puts you at a performance disadvantage before you start optimizing your application, so take it as a last resort.

Once you have the class definition, the work to customize the styles breaks down to writing your new styles and adding them to a class extension of the original custom element.

Your styles might look like this:

```css
// new-styles.css
:host {
    display: block;
}
div {
    color: blue;
    border: 1px solid;
    padding: 10px;
    margin: 10px;
    font-size: 20px;
    font-family: serif;
}
```
<dialog></dialog>

Those styles could be applied to the class extension via [Constructible Stylesheets](https://dev.to/westbrook/why-would-anyone-use-constructible-stylesheets-anyways-19ng) and [import assertions](https://tc39.es/proposal-import-assertions/):

```js
// NewCustomElement.js
import { CustomElement } from './CustomElement.js';
import styles from './new-styles.css' assert { type: 'css' };

export class NewCustomElement extends CustomElement {
    connectedCallback() {
        this.adoptedStyleSheets = [ styles ];
    }
}
```
<dialog></dialog>

And then you register the custom element with your new _styled_ definition:

```js
// new-custom-element.js
import { NewCustomElement } from './NewCustomElement.js';

customElements.define('custom-element', NewCustomElement);
```
<dialog></dialog>

And then you have:

<div>
    <custom-element>This is a styled custom element!</custom-element>
</div>

In the above code, we've removed all of the original styles for our customized styles, but that may not be the right call in a more complex custom element. You can apply your styles on top of what is already there via `this.adoptedStyleSheets = [ ...(this.adoptedStyleSheets || []), styles ];`. This would allow you to leverage the defaults for parts of the custom element that you do not want to take control of while placing your styles at the end of the cascade.

The `adoptedStyleSheets` API is still working towards full cross-browser availability (*cough* Safari *cough*), so it is also possible that the custom element you are working with does not, yet, apply styles in this way. The world of uncertainty that brings would benefit a custom solution, for what may be a _very_ custom element, so attempt the above with caution. One option would be to create a `<style>` tag with your new CSS and append it to the shadow root of the original custom element, which could cause some issues depending on the rendering strategy at play. In this way, were you not to want any of the existing styles of the original custom element, you could even `querySelectorAll('style')` and then remove them from the shadow DOM. Just keep in mind that the further into the past the techniques at play for applying styles to the custom element in question come from, the more risk that comes with this technique.

<script>
const template = document.createElement("template");
template.innerHTML = /*html*/`
<style>
    :host {
        display: block;
    }
    div {
        color: blue;
        border: 1px solid;
        padding: 10px;
        margin: 10px;
        font-size: 20px;
        font-family: serif;
    }
</style>
<div>
    <slot></slot>
</div>
`;
class CustomElement extends HTMLElement {
    constructor() {
        super();
        this.attachShadow({
            mode: "open"
        });
        this.shadowRoot.appendChild(
            template.content.cloneNode(true)
        );
    }
}
customElements.define('custom-element', CustomElement);
</script>
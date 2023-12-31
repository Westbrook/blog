---
layout: layouts/post.html
title: Styling custom elements with CSS Parts
date: 2023-12-31
---

{% block content %}
When you want _more_ power, but you don't wall _all_ the responsibility, you better hope that you're seated right next to the developer of the custom element to request updates or that they've already embued the custom element with a styling API. That's right, a custom element can have a styling API, one of which can come in a number/combination of forms:

- [CSS Custom Properties](/styling/css-custom-properties/), e.g. `--component-color: darkgray;`
- [CSS Parts](/styling/css-parts/), e.g. `::part(button) { color: darkgray; }`
- [Continer style query based themes](/styling/container-style-queries/): `custom-element { --custom-theme: exciting; }`
- Slotted content, including [stacked slots](/styling/stacked-slots/)

Let's dive into how we might leverage a styling API built of CSS Parts!

Starting without increadibly clever and complex custom element example:

```js
/* custom-element.js */
import styles from './custom-element.css' with { type: 'css' };
const template = document.createElement("template");
template.innerHTML = /*html*/`
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
        this.shadowRoot.adoptedStyleSheets = [styles];
    }
}

customElements.define('custom-element', CustomElement);
```

And its equally exciting and educational CSS:

```css
/* custom-element.css */
:host {
    display: block;
}
div {
    color: red;
    border: 1px solid;
    padding: 30px;
    font-size: 30px;
}
```

We've learned that there's not much to do about customizing these styles from the outside. However, what _might_ this look like with a CSS Part-based styling API? Luckily, in this case, not much would need to change. The styles would stay the same, as well as most of the custom element definition. However, there would need to be one small tweak in the template that was applied:

```js
/* custom-element.js */
import styles from './custom-element.css' with { type: 'css' };
const template = document.createElement("template");
template.innerHTML = /*html*/`
    <div part="container">
        <slot></slot>
    </div>
`;

// Class definition...
```

Here we resurface a single CSS Part called `container`, and for custom element this minimal, we'd be done. Now a direct consumer of the custom element in question would have full control over the shadow DOM element in question from the outside. That means applying the page styles we've seen in other demos would look like this:

```css
/* site.css */
::part(container) {
    color: green;
    border: none;
    padding: 1em 2em;
    font-size: 2rem;
}
```

You don't even _technically_ need to `custom-element` tag name selector to start. However, do keep in mind that `container` as a part name is pretty generic and that the `::part(...)` selector does reach into all custom elements with shadow roots in a sinlge DOM tree, so you may want to leverage higher selector specificity: e.g. `custom-element::part(container)`, or even something bound directly to a class or id on the element.

Another nice feature of CSS Parts is they also give you access to pseudo classes and elements on the element to which they map. That means you could further customize our `container` with some `:before` content:

```css
/* site.css */
::part(container) {
    color: green;
    border: none;
    padding: 1em 2em;
    font-size: 2rem;
}
::part(container):before {
    content: '🥳 ';
}
```

After which, we now have:

<div class="customized">
    <custom-element>A celebratory custom element!</custom-element>
</div>

It's certainly not the most beautiful custom element, yet... but it could be.

Whereas CSS Custom Properties would to give a very controlled API surface for specifically applies values in your styling API, CSS Parts applies a very free API to control specific DOM nodes in the shadow root. These techniques could be used individually or in combination with each other. They could even be stacked on top of each other so that a consumer of a custom element with such an API could choose to take more or less control of the styles applied to the element as needed by the parent context. As you learn to build and consume the styling APIs of custom elements, it's important to learn and understand the capabilities that each available technique surfaces.

<script type="module">
    const template = document.createElement("template");
    template.innerHTML = /*html*/`
        <div part="container">
            <slot></slot>
        </div>
    `;
    const styles = new CSSStyleSheet();
    styles.replaceSync(`
        :host {
            display: block;
        }
        div {
            color: red;
            border: 1px solid;
            padding: 30px;
            font-size: 30px;
        }
    `);
    class CustomElement extends HTMLElement {
        constructor() {
            super();
            this.attachShadow({
                mode: "open"
            });
            this.shadowRoot.adoptedStyleSheets = [styles];
            this.shadowRoot.appendChild(
                template.content.cloneNode(true)
            );
        }
    }
    customElements.define('custom-element', CustomElement);
</script>

{% endblock %}
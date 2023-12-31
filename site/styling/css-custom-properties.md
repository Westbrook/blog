---
layout: layouts/post.html
title: Styling custom elements with CSS Custom Properties
date: 2023-12-30
---

{% block content %}
When you want _more_ power, but you don't wall _all_ the responsibility, you better hope that you're seated right next to the developer of the custom element to request updates or that they've already embued the custom element with a styling API. That's right, a custom element can have a styling API, one of which can come in a number/combination of forms:

- [CSS Custom Properties](/styling/css-custom-properties/), e.g. `--component-color: darkgray;`
- [CSS Parts](/styling/css-parts/), e.g. `::part(button) { color: darkgray; }`
- [Continer style query based themes](/styling/container-style-queries/): `custom-element { --custom-theme: exciting; }`
- Slotted content, including [stacked slots](/styling/stacked-slots/)

Let's dive into how we might leverage a styling API built of CSS Custom Properties!

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

We've learned that there's not much to do about customizing these styles from the outside. However, what _might_ this look like with a CSS Custom Property-based styling API? Luckily, in this case, not much would need to change. The custom element definition and its DOM would stay the same. It would even be able to continue importing its styles from `custom-element.css`. However, the rules therein would likely look a little different:

```css
/* custom-element.css */
:host {
    display: block;
}
div {
    color: var(--custom-element-color, red);
    border: var(--custom-element-border, 1px solid);
    padding: var(--custom-element-padding, 30px);
    font-size: var(--custom-element-font-size, 30px);
}
```

Here we resurface a custom element for all of the styles that are applied by default, which is both a sort of maximalist _and_ minimalist approach to leveraging this form of styling API. The important thing to know here is that no matter how many CSS Custom Properties the element author surfaces for a consumer they will only ever be the values that the author choses to surface. The above example allows for altering or unsetting all of the rules applied by the element author, but no other styles. Additional styles could be included without fallbacks so as to not effect the default delivery of the custom element:

```css
/* custom-element.css */
:host {
    display: block;
}
div {
    color: var(--custom-element-color, red);
    border: var(--custom-element-border, 1px solid);
    padding: var(--custom-element-padding, 30px);
    font-size: var(--custom-element-font-size, 30px);
    font-weight: var(--custom-element-font-weight);
    margin: var(--custom-element-margin);
    background: var(--custom-element-background);
    height: var(--custom-element-height);
    width: var(--custom-element-width);
}
```

Or, even less of the applied styles could be surfaced for customization:

```css
/* custom-element.css */
:host {
    display: block;
}
div {
    color: var(--custom-element-color, red);
    border: 1px solid;
    padding: var(--custom-element-padding, 30px);
    font-size: 30px;
}
```

Here we see that the `border` and `font-size` are sacrosanct and not for customization within the boundaries of the surfaced API.

Regardless of the depth of customization surfaced in this way, a custom element author should be sure to include these values as actual API in their documentation and manage them with semver as with any other part of the element's API so that they can be productively relied on and leveraged over time. Doing so is almost as easy as what we saw in our initial customization of an element's delivery. In that case we applied the following styles at the page level:

```css
/* site.css */
custom-element {
    color: green;
    border: none;
    padding: 1em 2em;
    font-size: 2rem;
}
```

Taking the first example above, where the styling API only offers customization of the rules addressed directly by the author, we could acheive the same via:

```css
/* site.css */
custom-element {
    --custom-element-color: green;
    --custom-element-border: none;
    --custom-element-padding: 1em 2em;
    --custom-element-font-size: 2rem;
}
```

After which, we once again have:

<div class="customized">
    <custom-element>A styled custom element!</custom-element>
</div>

It's certainly not the most beautiful custom element, yet... but it could be. It's also not the most fluent of CSS Custom Property APIs. Building from the basics we've gone over here you can see this technique scale up to enormous levels of nuance and capability. Imagine a custom element with a styling API built around a system like [Open Props](https://open-props.style/). The world is your oyster, make yourself a pearl!

<script>
    const template = document.createElement("template");
    template.innerHTML = /*html*/`
        <div>
            <slot></slot>
        </div>
    `;
    const styles = new CSSStyleSheet();
    styles.replaceSync(`
        :host {
            display: block;
        }
        div {
            color: var(--custom-element-color, red);
            border: var(--custom-element-border, 1px solid);
            padding: var(--custom-element-padding, 30px);
            font-size: var(--custom-element-font-size, 30px);
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
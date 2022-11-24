It does seem like I enjoy a good `<slot></slot>`. I mean look, I wrote about them all the way back in 2018 in [`<slot/>`ing in Some Tips](https://medium.com/@westbrook/slot-ing-in-some-tips-4f2763fc2ca), and then in 2020, I spoke about [Stacked Slots](https://youtu.be/MS7y2K7tZto?t=3060) at a virtual Web Components SF meetup (see the [associated slides](https://webcomponents.dev/edit/5sXaRWYCLldZnVq9VjzT/src/code-example.ts)), before sharing a proof of concept for [Light DOM as Model](https://webcomponents.dev/edit/F4jBbQpeMSujg9FtRrtZ/src/light-dom-as-model.ts). And, as if that weren't enough, here we are again, and I'm writing to you, friend, about `<slot>`s. Today, we're going to get out of the theoretical and into the practical as we look at actual usage of Stacked Slots that I'm excited to bring to life as part of Adobe's [Spectrum Web Components](https://opensource.adobe.com/spectrum-web-components/) to support the delivery of Spectrum design's [Help Text pattern](https://spectrum.adobe.com/page/help-text/).

## Help text

![Spectrum help text pattern](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0r3wsbv3sm5rpt93h8y1.png)

Spectrum’s help text pattern:

> provides either an informative description or an error message that gives more context about what a user needs to input. It’s commonly used in forms.

Clearly, help text isn’t much use on its own. There needs to be some way to associate help text with the element it describes. Traditionally, that might look like:

```html
  <input aria-describedby="help-text" />
  <div id="help-text">
    The above input is described by this help text.
  </div>
```

However, we are gathered here today to celebrate the awesomeness that are [`<slot>`s](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/slot), so we have to be talking about [shadow DOM](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_shadow_DOM) (required to leverage browser native `<slot>` elements and their various capabilities), and are likely talking about [custom elements](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements) (because they all get along so well). With these two APIs together, it's not uncommon to see a form element writ into a custom element as follows:

```html
  <custom-form-element></custom-form-element>
```

From here, you're likely to see many APIs structured in a "component as function with properties" pattern that is common in various javascript frameworks and surface `help-text` as an attribute of the `<custom-form-element>`:

```html
  <custom-form-element
    help-text="An input inside of this element is described by this help text."
  ></custom-form-element>
```

You might then see such an API expanded for help text across various states:

```html
  <custom-form-element
    help-text="An input inside of this element is described by this help text."
    help-text-invalid="An input inside of this element is described by this help text when invalid."
  ></custom-form-element>
```

Possibly, ad infinitum:

```html
  <custom-form-element
    help-text="An input inside of this element is described by this help text."
    help-text-valid="An input inside of this element is described by this help text when valid."
    help-text-invalid="An input inside of this element is described by this help text when invalid."
    help-text-when-you-appear-stuck="An input inside of this element is described by this help text when you appear stuck."
    help-text-etc="An input inside of this element is described by this help text, etc."
  ></custom-form-element>
```

This API had only just started to talk about the `<custom-form-element>` and already it is quite thick (imagine it with actual API for customizing the associated form element). This could certainly weigh on your consuming developers. What's more, while this is going to look (and work) great when javascript is on, or look great (though possibly not work) when delivered in a browser with [Declarative Shadow DOM](https://web.dev/declarative-shadow-dom/), it has no chance of looking good with neither (unless _generally_ not seeing something counts as it looking good, which is true... _sometimes_), and it certainly doesn't use any `<slot>`s!

To be clear, none of the notes above are inherently bad. Each of these could align with the desired philosophy of an element, a library of elements, or an application that leverages elements. I call them out here as a way to get to the point I'd like to make about `<slot>`s. If your use case doesn't direct you towards an HTML-like API for your custom elements, more power to you. However, supplying this content as HTML allows us to:

- customize the DOM element that wraps your help text content
- deliver DOM in that content (e.g. anchor tags, icons, etc.)
- encapsulate any default functionality or styles belonging to help text content
- separate the concerns of delivering a form element from those of delivering help text content
- style help text content directly from the outside

To support all these things, I'd like to propose the following API.

```html
  <custom-form-element>
    <custom-help-text slot="help-text">
      An input inside of this element is described by this help text.
    </custom-help-text>
    <custom-help-text slot="negative-help-text">
      An input inside of this element is described by this help text when invalid.
    </custom-help-text>
  </custom-form-element>
```

The above makes the presence and the customization of content beyond the initial help message easy to manage regardless of the context from which you're delivering it. You've got a custom form element; you slot in the default and negative help text messages and automatically they are correctly associated with the appropriate element within the parent's shadow DOM and hidden/shown based on the validity of the said parent. Similarly, you can slot in a single piece of help text, and it can be fully controlled from the JS scope in which is delivered:

```html
  <custom-form-element>
    <custom-help-text slot="help-text">
      An input inside of this element is described by this help text.
    </custom-help-text>
  </custom-form-element>
```

Let's look at how we can bring all of this into being.

## &lt;slot/&gt;ing in some help text

For this last example, a single slot named `help-text` is all you need to get started. Here's what that could look like in a no dependency custom element where we're taking nothing else into account except slotting in this one piece of content:

```js
const template = document.createElement("template");
template.innerHTML = /*html*/`
  <input />
  <slot name="help-text"></slot>
`;

class CustomFormElement extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: "open" });
    this.shadowRoot.appendChild(template.content.cloneNode(true));
  }
}

customElements.define("custom-form-element", CustomFormElement);
```

If this already has you excited to code, hop on over to [webcomponents.dev]
(https://webcomponents.dev/) and [fork the project from this point]
(https://webcomponents.dev/edit/sTddx519tvSvSKT0984P?branch=00-slot%40wfVpXdsTYhf9mY2slxW4t2Y2SjC3).

What we've got here _is_ relatively simple to start, so we've got lots of growing to do. Before we dive into supporting the `nuetral-text` and `negative-text` slots, where we can really dig into the concept of Stacked Slots, let's be sure that the content in this slot can be appropriately associated with the `<input />` element that we're basing our `<custom-form-element>` around for the time being.

All form controls need to be supplied with a label (something we are actively omitting at this time) to be accessible. Labels can be associated with a form control as follows:

- the form control can supply this content itself via the `aria-label` attribute
- a secondary element can be referenced by ID in the form control's `aria-labelledby` attribute, or
- the form control itself can be referenced by ID in a `<label>` element's `for` attribute

Content associated in this way will be read as part of the primary description of the form control. Help text content doesn't require this level of priority in the element's description, so we will associate it by leveraging the form control's `aria-describedby` attribute to reference this content by ID.

When referencing content by ID, it is important to remember that the elements on either side of the reference need to share a DOM tree for the reference can be completed. With our form control (the `<input />`) being within our shadow root and our `<custom-help-text>` element being slotted from the outside we can't simply add an ID on one and point to it from the other. Instead, we'll place our `<slot>` element into a `<div>` itself and give that `<div>` the ID to reference from our form control. In this way, the `<div>` can adopt the help text content projected onto the `<slot>` element and make it available to the form control to reference via ID with its `aria-describedby` attribute.

```js
template.innerHTML = /*html*/`
  <input aria-describedby="help-text" />
  <div id="help-text">
    <slot name="help-text"></slot>
  </div>
`;
```

[See it in living color](https://webcomponents.dev/edit/sTddx519tvSvSKT0984P?branch=01-associated%40wfVpXdsTYhf9mY2slxW4t2Y2SjC3), fork it, share it, and when you're done, come right back, so we can dig even deeper.

## Now for the superpowers

That's right, custom elements allow you to give your HTML superpowers. A consumer of our element could already take what we've made, leverage the bubbling and composed events out of the `<input />` element, and manage the text that is supplied to the `<custom-help-text>` element from the outside, and we'll see what that looks like shortly. However, we can save most consumers a lot of work by baking some basic management directly into our custom element. This is exactly what is outlined in the code example above:

```html
  <custom-form-element>
    <custom-help-text slot="help-text">
      Describe interests you would like to explore.
    </custom-help-text>
    <custom-help-text slot="negative-help-text">
      Enter at least one interest.
    </custom-help-text>
  </custom-form-element>
```

While there is much that can be done about managing when something deserves "negative help text", for our demo we'll only track whether our `<input />` has content or not. To do so, and rather naively at that, we'll add the `required` attribute. This will allow us to leverage the `checkValidity()` method on the `<input />` element to confirm whether any content has been supplied.

To support this, we'll expand our `constructor` to bind a listener for the `input` event to our host element:

```js
constructor() {
  super();
  this.attachShadow({ mode: "open" });
  this.shadowRoot.appendChild(template.content.cloneNode(true));
  this.addEventListener('input', this.handleInput);
}
```

Adding the listener to `this`, even though the `input` occurs within our shadow DOM, is useful to ensure that the `handleInput()` callback is bound to our host element, and not the child `<input />`. In our example, we'll be keeping the held state to an absolute minimum, but having `this` reference the host element not only allows us to directly query selectors within our element, were we to hold state on our element we'd have access to that as well.  In the `handleInput()` callback we can examine the `event`'s `composedPath()` to reacquire the `<input />` element and call `checkValidity()` on it:

```js
handleInput(event) {
  // ...
  const target = event.composedPath()[0];
  const invalid = !target.checkValidity();
  // ...
}
```

We'll leverage the `invalid` variable to tell our element which of the slotted content to accept at any one time. I chose to use `invalid` here just in case we promote this to an attribute on our element in the future. By default, a boolean attribute should be `false`, which means the attribute is absent from the element so that the element isn't sprouting attributes on every construction. Tracking an element that is "valid" by default with an attribute or property of `invalid` aligns with this pattern and reduces possible render/paint side effects.

While we're here, we will supply a pass through getter to allow access to the value of our `<input />` from the outside:

```js
  get value() {
    return this.shadowRoot.querySelector('input').value;
  }
  
  set value(value) {
    this.shadowRoot.querySelector('input').value = value;
  }
```

Next we'll get into what we actually do with these values.

### All the slots

So far, we've discussed the consumption of two different slots: `help-text` and `negative-help-text`. Why?

Well, some consumers want all the control. In this use case, we surface the `help-text` slot so that absolutely anything can be put into it, whenever the parent application would like. Want to cheer your visitor on for every keystroke they make, this is the slot for that.

```html
  <custom-form-element
    oninput="
      const modiferElement = this.querySelector('#modifier');
      const countElement = this.querySelector('#count');
      const multipleElement = this.querySelector('#multiple');
      const length = this.value.length;
      multipleElement.textContent = length === 1
        ? ''
        : 's';
      countElement.textContent = length;
      let modiferText = '';
      if (length > 10) {
        modiferText = 'Wow!';
      } else if (length > 5) {
        modiferText = 'Nice.';
      } else if (length > 0) {
        modiferText = 'Keep going.';
      }
      modiferElement.textContent = modiferText;
    "
  >
    <custom-help-text slot="help-text">
      <span id="modifier"></span> You've typed <span id="count">0</span> character<span id="multiple">s</span>.
    </custom-help-text>
  </custom-form-element>
```

Really, with the `help-text` slot, [access to the form control's value](https://webcomponents.dev/edit/sTddx519tvSvSKT0984P/src/index.js?branch=02-value%40wfVpXdsTYhf9mY2slxW4t2Y2SjC3), and experience with which to interact with it from the outside, there's no end to how you could leverage the content you might supply. However, in more cases than not, swapping between "this is what you should do" and "this is how you get out of the problem you've gotten yourself in" text will likely support the goals of our consumers. In some cases, we might just be turning on the "this is how you get out of the problem you've gotten yourself in" text. Pairing the `help-text` slot with a `negative-help-text` slot, we can make the process of doing these things something they'll _almost_ never have to think about.

So, where are we actually inserting these slots into our element's shadow DOM? We might start by positioning them next to each other, as siblings:

```js
template.innerHTML = /*html*/`
  <input aria-describedby="help-text" />
  <div id="help-text">
    <slot name="help-text"></slot>
    <slot name="negative-help-text"></slot>
  </div>
`;
```

From here we could leverage the `invalid` variable we've already derived in our `handleInput()` method to do something like add the `hidden` attribute (ignoring that [[hidden] is a lie](https://meowni.ca/hidden.is.a.lie.html)) to the `<slot>` elements conditionally. Then we can show the `negative-help-text` slot when `invalid === true`. When `invalid === false`, we can show the `help-text` slot. And, we're done.

Except, what happens when content is only addressed to the `help-text` slot?

Toggling `hidden` directly on both `<slot>` elements in response to `invalid` would mean that the content addressed to the `help-text` slot would be hidden when `invalid`, regardless of whether there was content to display in the `negative-help-text` slot at that time. We could _only_ toggle `hidden` on the `negative-help-text` slot, but that would mean that there are times that our `<custom-from-element>` would receive two pieces of help text. Some component authors might want to deliver exactly this functionality to their users, in which case, they'll be ready to go with the above. For those of you that would agree, [here's your off ramp](https://webcomponents.dev/edit/sTddx519tvSvSKT0984P/stories/index.stories.js?branch=03-both@wfVpXdsTYhf9mY2slxW4t2Y2SjC3).

For those of you who, like me, see more than one type of help text as something to prevent, there are a couple of options available. One would be to leverage the `slotchange` event, and the `assignedElements()` API on the `negative-help-text` slot to decide whether it has content, and when it does use that state in concert with `invalid` to decide when to hide the `help-text` slot. One more event listener, one more callback method, one quick question about `slotchange` timing and whether you should hold state instead, and you'd be ready to go! But, what if I told you that the browser already had this functionality built directly into it?

Well, it does.

### Stacked slots

A `<slot>` element doesn't just act as a marker for where content can be projected into your element from the outside. The `<slot>` element can also supply default content to be delivered when there is nothing projected onto it, as well. What's more, that content can be additional `<slot>` elements. In this way, we can stack our slots and give the earliest ancestor `<slot>` in the stack precedence over those that descend from it. For our help text slots, that could look like:

```js
template.innerHTML = /*html*/`
  <input aria-describedby="help-text" />
  <div id="help-text">
    <slot name="negative-help-text">
      <slot name="help-text"></slot>
    </slot>
  </div>
`;
```

This allows any content addressed to the `negative-help-text` slot to "win" and be the content that is shown when it is available. When that is absent, content addressed to the `help-text` slot will always be available for users to manage directly from the outside. That means that when HTML like the following is used, only the "This field is required!" content that is addressed to the `negative-help-text` slot will be displayed on the rendered page.

```html
  <custom-form-element>
    <custom-help-text slot="help-text">Please type something here.</custom-help-text>
    <custom-help-text slot="negative-help-text">This field is required!</custom-help-text>
  </custom-form-element>
```

This isn't exactly what we were looking for, so we need to add a little something more. We want out parent `<slot>` to pass through to our child `<slot>` when the form control's value is valid. To do this, instead of relying on `hidden` to show or hide the element, we can use a nonsensical `name` to ensure content can't be addressed to the slot.

```js
template.innerHTML = /*html*/`
  <input aria-describedby="help-text" />
  <div id="help-text">
    <slot
      name="pass-through-help-text-${Math.random()}"
      id="negative-help-text"
    >
      <slot name="help-text"></slot>
    </slot>
  </div>
`;
```

This will be our default, allowing the `help-text` slot to take precedence. While handling the `input` event we can then use the value of `invalid` to toggle the `name` attribute to something addressable.

```js

handleInput(event) {
  // ...
  const target = event.composedPath()[0];
  const invalid = !target.checkValidity();
  const slot = this.shadowRoot.querySelector('#negative-help-text');
  slot.name = invalid
    ? 'negative-help-text
    : `pass-through-help-text-${Math.random()}`;
}
```

This will give our content addressed to `negative-help-text` a slot onto which to be projected when out form control becomes `invalid` without preventing content addressed to the `help-text` slot from being displayed when `negative-help-text` content is absent. It's a bit like API validation in HTML.

With all of this together, we'll be imbuing our `<custom-form-element>` with the following super powers:

- accessible help text content
- a `help-text` slot to act as default and receive updates from the outside while displaying in all validity states
- a conditional `negative-help-text` slots for overriding that content with content meant only for when the form control is `invalid` 

It is a neat little custom element who's definition looks about like:

```js
const template = document.createElement("template");
template.innerHTML = /*html*/`
  <input
    aria-describedby="help-text"
    required
  />
  <div id="help-text">
    <slot
      id="contextual-help-text"
      name="pass-through-help-text-${Math.random()}"
    >
      <slot name="help-text"></slot>
    </slot>
  </div>
`;

class CustomFormElement extends HTMLElement {
  get value() {
    return this.shadowRoot.querySelector('input').value;
  }
  
  set value(value) {
    this.shadowRoot.querySelector('input').value = value;
  }

  constructor() {
    super();
    this.attachShadow({ mode: "open" });
    this.shadowRoot.appendChild(template.content.cloneNode(true));
    this.addEventListener('input', this.handleInput);
  }

  handleInput(event) {
    const contextualHelpTextSlot =
      this.shadowRoot.querySelector('#contextual-help-text');
    const target = event.composedPath()[0];
    const invalid = !target.checkValidity();
    contextualHelpTextSlot.name = invalid
      ? 'negative-help-text'
      : `pass-through-help-text-${Math.random()}`;
  }
}

customElements.define("custom-form-element", CustomFormElement);
```

[Check it out more closely](https://webcomponents.dev/edit/sTddx519tvSvSKT0984P/src/index.js?branch=04-contextual%40wfVpXdsTYhf9mY2slxW4t2Y2SjC3&p=stories), as well as demos for working with the `help-text` and `negative-help-text` slots to varying levels of complexity.

## What's missing?

Before we wrap up, let's visit a (not so short) list of things that were consciously omitted from the conversation in this article. If you find any that I've unconsciously missed, or if one of these is of particular interest to you, please let me know in the comments below as I'd want to get it added when missing or take your interest as a nice push towards any sequel posts with which I might follow this.

The first to come to mind is giving a definition to our `<custom-help-text>` element that is features throughout our demos but until this point has no superpowers of its own.

Revisiting our screenshot of what help text could look like, we've clearly skipped:

- labeling our form element
- supporting API like `required` (see the * mark) on `<custom-form-element>`
- styling content that we build this way. Yes, I noticed that the visuals in that image are much more appealing than our demos to date

Thinking about the validity and value piping that we added:

- what other APIs (beyond `value`) of our form element should be passed through to our `<custom-form-element>`?
- what other forms of validity (beyond `required`) should be available and how can we surface them?
- how can we make this validity and value a part of a parent `<form>` element's form data? (Yes, the shadow DOM boundary effectively hides the `<input />` from participating in our custom element's current form.)

Looking at how we related our help text to our form control, we might also be interested in:

- supporting form controls that are outside of our shadow root
- abstracting this relationship into a reusable form that could empower many custom form control elements, rather than just one
- the benefits of simplifying this code with the help of a declarative templating system

Clearly, there is a lot that we could do to continue to expand on what we've started here together. Some might even involve more learning and praise around `<slot>` elements, but most spread across the breadth of APIs and capabilities that are available in the browser today as web components.

But, just because there is a lot to do, doesn't mean we haven't already achieved a lot together. 

## How far we've come

![A screen shot of all of the stories we've created.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pzrdrxqzeko9jpq3zp7c.png)

At press time, [our demo](https://webcomponents.dev/edit/sTddx519tvSvSKT0984P/src/index.js?branch=02-validity@wfVpXdsTYhf9mY2slxW4t2Y2SjC3) features:

- a reusable `<custom-form-element>` element
- a single `required` `<input />` element on which we track validity
- piping for seeing the value of the `<input />` from the outside
- the `<input />` is accessibly related to "help text" content that can be supplied from the outside as HTML
- a default `<slot>` (`help-text`) surfaces a wide array of from the outside customizations around the delivery of the help text 
- an override `<slot>` (`negative-help-text`) that is available when the `<custom-from-element>` is `invalid`

To do so we've learned about:

- building a custom element from scratch
- how amazing [webcomponents.dev](https://webcomponents.dev/) is
- ways to manage property/attribute bloat
- accessibly relating content from the outside of a custom element to content with that element's shadow DOM
- bind events to custom elements
- some basics around boolean attributes
- making state on elements encapsulated within our shadow root public
- `<slot>` elements and their default content
- stacking `<slot>` elements

Hopefully, you've found it interesting and educational. Feel free to share anything I could be more clear about in the comments below so that this content can be even more useful for our readers.

After that, you can visit a more advanced version of this technique where it is being leveraged in [Spectrum Web Components](https://opensource.adobe.com/spectrum-web-components/components/help-text/). There you'll see this functionality delivered as a class factory mixin into custom elements based on the [lit](https://lit.dev) library and their [`LitElement`](https://lit.dev/docs/api/LitElement/) base class.

---

Photo by <a href="https://unsplash.com/@aarez?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Aarón González</a> on <a href="https://unsplash.com/s/photos/slot?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  

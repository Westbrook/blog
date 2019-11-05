> _Disclaimer: it was brought to my attention that in my desire to strike a very click-bait-like pose in reference to a wide field of "considered harmful" articles, that it might too directly call to mind a [seminal post](https://reactjs.org/blog/2016/07/13/mixins-considered-harmful.html) in this regard that takes a clear stance in opposition to its subject. While it is absolutely my goal to trick you onto this page with such a title, I won't pretend to have THE definitive stance on almost any subject, let alone on one as rich and varied as event handling. I do hope to strike up a good dialog if you'll join me in one, and find that a shared foundation of knowledge is the best place to get started on one. So, let's begin!_ 

First of all, what even is `composed: true`, and when _might_ you use it?

[`Event.composed`](https://developer.mozilla.org/en-US/docs/Web/API/Event/composed) outlines whether a DOM event will cross between the shadow DOM in which the event is dispatched into the light DOM in which the element that the shadow root is attached to exists. As you'll find in the MDN article on the subject, "all UA-dispatched UI events are composed" by default, but when you work with manually dispatched events you have the opportunity to set the value for this property as you see fit. So the "what" of `composed: true` at its simplest is "a way to manage the encapsulation of your event transmission", and the "when" is namely "while working with shadow DOM", a practice that while being not exclusive to has become somewhat synonymous to working with web components; shadow DOM, custom elements, ES6 modules, and the `<template>` element. Next, we'll review some important concepts before we try to come to a decision about `composed: true`:

- [Native DOM events and how they work](#native-elements)
- [Manually dispatched events and their configurations/extensions](#manual-dispatch)
- [The `detail`s on Custom Events](#details)
- [The world of events within a shadow root](#shadow-root)
- [Composed Events](#composed)

At that point, we'll all be specialists and we can get into some practices and patterns with DOM events that might be useful in your applications. I'll share some ideas that I've had or used, and I hope you'll do the same in the comments below. Ready to go?

## Native DOM Events <a id="native-elements"></a>

Native HTML elements communicate up the DOM tree using DOM events. You might be used to seeing this with elements like `<input />` which publish events like `change` and `input` or with the `<button />` element, where it's common to rely on the `click` event that it publishes. It might not be immediately clear you are relying on these things, but if you've worked with `onclick` (native) or `onChange` (virtual DOM) properties, it is these DOM events that you are relying on under the hood. Knowing that these events are dispatch along the DOM tree, we can choose locations (either explicit or general) at which to listen for the via the `addEventListener(type, listener[, options/useCapture])` method that is present on any `HTMLElement` based DOM node.

These events have two phases; the "capture" phase and the "bubble" phase. During the capture phase, the event travels from the top of the DOM down towards the dispatching element and can be listened for on each of the elements that it passes through in this phase by setting the third argument of `addEventListener()` to true, or by explicitly including `capture: true` in an `options` object passed as the third argument. For example the steps of the "capture" phase of a `click` event on the `<button>` in the following DOM structure:

```html
<body>
    <header>
        <nav>
            <button>Click me!</button>
        </nav>
    </header>
</body>
```

Would be as follows:

1. `<body>`
2. `<header>`
3. `<nav>`
4. `<button>`

Then, being a `click` event has `bubbles: true` by default, the event would enter the "bubble" phase of the event and travel back up the DOM passing through the above DOM in the following order:

1. `<button>`
2. `<nav>`
3. `<header>`
4. `<body>`

At any point in either phase that you are listening for this event, you will have access to the `preventDefault()`, `stopPropagation()`, and `stopImmediatePropagation()` methods that give you powerful the events that travel across your application. `preventDefault()` can most clearly be felt when listening to a `click` event on an `<a href="...">` tag. In this context, it will _prevent_ the anchor link from being activated and prevent the page from navigating. In a way, this is the event asking for permission to do an action, and we'll look at this more closely in conjunction with manually dispatched events. `stopPropagation()` prevents the event in question from continuing along the DOM tree and triggering listeners along that path, a sort of escape valve for the event when certain parameters are met. This can be taken one step further via `stopImmediatePropagation()` which also prevents the event from completing the current step of the phase it is in. This means that no later bound listeners on that same DOM element for the event in question will be called. So for the `<button>` element in the example above, when a `click` event is dispatched, you could imagine the following completely trivial listeners:

```js
const body = document.querySelector('body');
const header = document.querySelector('header');
const button = document.querySelector('button');
// You can hear the `click` event during the "capture" phase on the `<body>` element.
body.addEventListener('click', () => {
    console.log('heard on `body` during "capture"');
}, true);
// You cannot hear the `click` event during the "bubble" phase on the `<body>` element.
body.addEventListener('click', () => {
    console.log('not heard `body` during "bubble"');
});
// You can hear the `click` event during the "bubble" phase on the `<header>` element.
header.addEventListener('click', (e) => {
    console.log('heard on `header` via listener 1 during "bubble"');
    e.stopPropagation();
});
// You can hear the `click` event during the "bubble" phase on the `<header>` element.
header.addEventListener('click', (e) => {
    console.log('heard on `header` via listener 2 during "bubble"');
    e.stopImmediatePropagation();
});
// You cannot hear to the `click` event during the "bubble" phase on the `<header>`
// element being it is bound later than the previous listener and its use of the
// `stopImmediatePropagation()` method.
header.addEventListener('click', (e) => {
    console.log('not heard on `header` via listener 3 during "bubble"');
});
// You can hear the `click` event during the "capture" phase on the `<button>` element.
button.addEventListener('click', () => {
    coonsole.log('heard on `button` during "capture"');
}, true);

button.click();
// heard on `body` during "capture"
// heard on `button` during "capture"
// heard on `header` via listener 1 during "bubble"
// heard on `header` via listener 2 during "bubble"
```

The majority of values for `bubbles`, `cancelable` (needed to empower `preventDefault()`), and `composed` are the same across native DOM events, and in many of those cases the value of `composed` is `true`, so it's possible that the browser is already refuting the idea that it could "harmful". However, when working with native DOM events the values for these three properties are also not configurable. To access the power, and responsibility, that comes with being able to do so, you'll need to enter the world of manually dispatched events.

## `dispatchEvent()` <a id="manual-dispatch"></a>

So far we've mainly talked about the `click` event as automatically dispatched by the browser. There is, of course, a whole family of UA-dispatched UI events that can be addressed in the same manner (e.g. `animationend`/`copy`/`keydown`/`mouseover`/`paste`/`touch`, etc.). However, the real fun starts when you take that power into your own hands and start dispatching events on your own creation. For this, the browser supplies us with the [`dispatchEvent()`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/dispatchEvent) method that hangs off of anything extended from `EventTarget`, which includes all of the `HTMLElement` based class of DOM elements. For this to do its magic we need to supply it an event to dispatch. We're given a number of events classes to create our new event from (e.g. `new Event()`, `new MouseEvent()`, `new InputEvent()`, etc.), but for the most part working with `new Event(typeArg[, initDict])` only is more than enough. 

Now, we're ready to dispatch an event.

```js
el.dispatchEvent('test-event');
```

Event dispatched!

The event has a `type` of `test-event`, so a listener like the following will be able to hear it:

```js
el.addEventListener('test-event', (e) => console.log(e.type));
// test-event
```

You can also listen for this event during the "capture" phase:

```js
const body = document.querySelector('body');
body.addEventListener('test-event', (e) => console.log(e.type), true);
// test-event
```

But, you won't be hearing it in the "bubble" phase:

```js
const body = document.querySelector('body');
body.addEventListener('test-event', (e) => console.log(e.type));
// ... ... Bueller?
```

This is because by default a `new Event()` (as well as all derivative event constructors) have `bubbles`, `cancelable`, and `composed` set to `false` by default. This is where the optional `initDict` argument of our event constructor comes into play. When you want to customize the values of these, you'll create your event like so:

```js
const event = new Event('test-event', {
    bubbles: true,
    cancelable: true,
    composed: true,
};
```

Or however best supports (or least harms? 😉) the use case in question. That means that if you only want your event to be available in the "capture" phase (which literally means it takes half the time for it to run synchronously through your application that it would if it were to also make a pass through the "bubble" phase) you can leave that out. Don't have an action that you'd like permission to do? You can leave out `cancelable`, too. Don't have shadow DOM? Decided definatively that `composed: true` is harmful? It's your rodeo, leave it out!

### Preventing Default

Being able to prevent default on a manually dispatched event is awesome. It allows you to structure the actions you dispatch across your application as permission gates. Your event is asking "do I have permission to do this thing?", and whether the answer to that question can be found nearby or far you'll be able to respond to that information as you see fit. Returning to our completely trivial sample DOM:

```html
<body>
    <header>
        <nav>
            <button>Click me!</button>
        </nav>
    </header>
</body>
```

Our button might want to dispatch a `hover` event with `cancelable: true` to ensure that in the current viewing context (as managed in a more central location) is an acceptable one for displaying `hover` content or making hover related visuals, like maybe certain mobile browsers aught to do so we don't have to tap twice to get the actual link action to work... In this case, the application manager attached to the `<body>` element will not grant permission to continue with this action:

```js
body.addEventListener('hover', e => e.preventDefault());
const event = new Event('hover', {
    bubbles: true,
    cancelable: true
});
const applyDefault = button.dispatchEvent(event);
console.log(applyDefault);
// false
console.log(event.defaultPrevented);
// true
```

Not only do we see this pattern in the native anchor tag, but you'll likely have noticed it in the various keyboard events, as well as many others. With `cancelable: true` in your can more closely follow the patterns and practices outlined by the browser.


## The `detail`s on Custom Events <a id="details"></a>

The ability for an event to outline that something _did_ or is _about to_ happen is a superpower in and of itself. However, there are cases when we want to know more than can be communicated via access to `e.target` (a reference to the dispatching element), we want to know it more clearly, or we want the dispatching element to receive access to information only available to the listening element. For this, the off-the-shelf event constructors for native UI events won't be enough. Luckily, we have two really great options to work with when this is the case: `new CustomEvent()` and `class MyEvent extends Event {}`.

### CustomEvent

`new CustomEvent(typeArg[, initDict])` can be used in your application exactly like any of the previous constructors we've discussed and is sometimes discussed as "the" interface by which to create manually dispatched events for its clever naming as a "custom" event. However, the real power that this constructor gives you is the inclusion of the `detail` property on the `initDict`. While `detail` isn't directly writable, it can be set to an object or an array that won't lose identity when being mutated by the listener. This means that not only can you append data to it when dispatching an event, you can also append/edit data in it at the listener, allowing you to use events to resolve the value of data manages higher in your application. Get ready for another trivial example by imagining the following HTML:

```html
<body>
    <header> ... </header>
    <main>
        <section>
            <h1>Resolving title...</h1>
            <h2>Resolving title...</h2>
        </section>
    </main>
</body>
```

From here text for our `<h1>` could be resolved a la:

```js
body.addEventListener('title', e => e.detail.tile = 'Hello, World!');
const event = new CustomEvent('title', {
    bubbles: true,
    detail: {
        title: 'Failed to find a title.'
    }
});

h1.dispatchEvent(event);
h1.innerText = event.detail.title;
```

This all comes to pass thanks to the availability of the `detail` property on the `initDict` for `new CustomEvent()` and the reality that DOM events are synchronous, which can be super powerful.

### Extending Event

A very similar, and much more in-depth, form of customization can be had from extending the `Event` base class. Immediately, this approach allows you to access data that you would hang off of the event without the intervening `detail`. However, the ability to use `instanceof` is where this approach really differentiates itself. Returning to the HTML in the example above, let's not resolve the values for both of the headline elements:

```js
class H1Title extends Event {
    constructor(title = 'Failed to find a title.') {
        super('title', {
            bubbles: true
        });
        this.title = title;
    }
}
class H2Title extends Event {
    constructor(title = 'Failed to find a title.') {
        super('title', {
            bubbles: true
        });
        this.title = title;
    }
}
body.addEventListener('title', e => {
    if (e instanceof H1Title) {
        e.title = 'Hello, World!';
    } else if (e instanceof H2Title) {
        e.title = 'We're going places.';
    }
});

const h1Title = new H1Title();
const h2Title = new H2Title();

h1.dispatchEvent(event);
h1.innerText = event.title;

h2.dispatchEvent(event);
h2.innerText = event.title;
```

Whichever approach you take, using DOM events to pass actual data around your application can be very powerful. For more information on leveraging events in this way, check out:

{% youtube 6o5zaKHedTE %}

## Events from the Shadow Root <a id="shadow-root"></a>

Up to this point, every event that we have discussed has been dispatched in a document without any shadow roots. Because of this, there have been no extenuating encapsulations to take into consideration meaning unless you were to leverage `stopPropagation()` or `stopImmediatePropagation()` on one of those events the "capture" phase would span the entire DOM tree from `document` to the dispatching element, and when `bubbles: true` the "bubble" phase would do the same in reverse. When attached to an element, a shadow root creates a sub-tree of DOM that is encapsulated from the main documents DOM tree. As discussed before, the majority of UA-dispatched UI events have `composed: true` by default and will pass between the sub-tree to the main tree at will. Now that we know how to manually dispatch events, we get to choose whether that is true about the events we create.

Before we do that, being it will happen a lot, let's take a look at what happens when an event with `composed: true` is dispatched within a shadow root. Take, for example, a `click` event (which also has `bubbles: true` by default) as triggered by the `<button>` in the following DOM tree:

```html
<document>
    <body>
        <div>
            <shadow-root-el>
                #shadow-root
                    <div>
                        <button>
                            Click here!
                        </button> <!-- click happens here -->
                    </div>
            </shadow-root-el>
        </div>
    </body>
</document>
```

As with an event in the light DOM, the `click` event here will begin its "capture" phase at the `<document>`. However, it's here that the first difference between light DOM and shadow DOM events will become clear, the `target` of this event will not be the `<button>` element, as the shadow root on `<shadow-root-el>` is designed to do, it will have encapsulated the DOM inside of its sub-tree and hidden it away from the implementing document. In doing so, it will have retargeted the event in question to the `<shadow-root-el>` instead.

```html
<document> <!-- event: `click`, phase: "capture", target: `shadow-root-el` -->
    <body>
        <div>
            <shadow-root-el>
                #shadow-root
                    <div>
                        <button>
                            Click here!
                        </button> <!-- click happens here -->
                    </div>
            </shadow-root-el>
        </div>
    </body>
</document>
```

The event will capture down the DOM tree with these settings until it enters the shadow root where we'll experience the next difference between light DOM and shadow DOM events. The shadow root is the first node in our sub-tree that encapsulates the internals of `<shadow-root-el>` meaning we are _inside_ of the encapsulated DOM and the internals are no longer obfuscated from us. Here the `target` will be the `<button>` element on which the `click` event explicitly occurred. 

```html
<document>
    <body>
        <div>
            <shadow-root-el>
                #shadow-root <!-- event: `click`, phase: "capture", target: `button` -->
                    <div>
                        <button>
                            Click here!
                        </button> <!-- click happens here -->
                    </div>
            </shadow-root-el>
        </div>
    </body>
</document>
```

From here, the event, still being in its "capture" phase, will continue to travel down the DOM until it reaches its `target` the `<button>`. Here it will be available in the "capture" phase, then as the first step of the "bubble" phase before traveling back up the DOM. 

```html
<document>
    <body>
        <div>
            <shadow-root-el>
                #shadow-root
                    <div>
                        <button>
                            <!-- event: `click`, phase: "capture", target: `button` -->
                            <!-- event: `click`, phase: "bubble", target: `button` -->
                            Click here!
                        </button> <!-- click happens here -->
                    </div>
            </shadow-root-el>
        </div>
    </body>
</document>
```

During the "bubble" phase the same effects of encapsulation that the event experienced in the "capture" phase will be in play, so while the target when the event passes the shadow root will be the `<button>` starting at the `<shadow-root-el>` the event will be retargeted to that element before continuing to bubble up the DOM.

```html
<document>
    <body>
        <div>
            <shadow-root-el> <!-- event: `click`, phase: "bubble", target: `shadow-root-el` -->
                #shadow-root <!-- event: `click`, phase: "bubble", target: `button` -->
                    <div>
                        <button>
                            Click here!
                        </button> <!-- click happens here -->
                    </div>
            </shadow-root-el>
        </div>
    </body>
</document>
```

### Extended Retargeting

When working with nested shadow roots (e.g. custom elements with custom elements inside of them) this event retargeting will happen at each shadow boundary that the event encounters. That means that if there are three shadow roots that the event passed through the `target` will change three times:

```html
<body> <-- target: parent-el -->
    <parent-el> <-- target: parent-el -->
        #shadow-root <-- target: child-el -->
            <child-el> <-- target: child-el -->
                #shadow-root <-- target: grandchild-el -->
                    <grandchild-el> <-- target: grandchild-el -->
                        #shadow-root <-- target: button -->
                            <button> <-- target: button -->
                                Click here!
                            </button> <!-- click happens here -->
                    <grandchild-el>
            <child-el>
    <parent-el>
</body>
```

This is, of course, one of the benefits of the encapsulation that a shadow root can provide, what happens in the shadow root stays in the shadow root and beyond the shadow root, a less direct version of those actions are available.

### The Composed Path Less Traveled

There are times when we need a look into that dirty laundry to get a peek at just where that event came from, be it `<button>`, `<div>`, `<a>`, or something else (it's hopefully a `<button>` or `<a>`...a11y, people!), and for those times we've got the `composedPath()` method on our events. At any point in the event's lifecycle, calling `composedPath()` on that event will give you an array of all the DOM elements on which it will be heard. The array is listed in "bubble" order, so the zeroeth item will be the dispatching element and the last item will be the last element through which the event will pass. That means you can always use the following code to ascertain the original dispatching element and outline the path along which the event will trave, assuming the previous example HTML:

```js
const composedPath = e.composedPath()
const originalDispatchingElement = composedPath[0];
console.log(composedPath);
// [
    button,
    document-fragment,
    grandchild-el,
    document-fragment,
    child-el,
    document-fragment,
    parent-el,
    body, html,
    document,
    window
]
```

It's here in `composedPath()` that the effects of `composed: true` are most clearly felt. When an event has `composed: true` that path will leaf from the original dispatching element all the way to the `window` that holds the entire `document`, but when an event has `composed: false` that path will end at the shadow root that contains the dispatching element. 

## Decomposing an Event <a id="composed"></a>

As we've seen so far, what `composed: true` does for an event is make it act as much like a native DOM event as possible by allowing its "capture" phase to start at the very root of the document (as well as across intervening shadow boundaries) and travel into the shadow DOM sub-tree where the original dispatching element lives before allowing the "bubble" phase to do the same in reverse. Along that path, the event will be further affected by the shadow roots that it passes through by having itself retargeted to the element on which that shadow root is attached. There is one more place where a `composed: true` event in a shadow root will perform differently than when not in one. `composed: true` allowing that event to cross the shadow root, it will fire (as if in the "bubble" phase, but without traveling up the DOM) on the element to which the shadow root is attached. That means that while a `composed: true, bubbles: false` event that was dispatched on `<event-dispatching-element>` would pass through all of the elements in the following code during the "capture", only the `<shadow-root-el>` would experience that event during the "bubble" phase.

```html
<div>
    <shadow-root-el>
        #shadow-root
            <section>
                <div>
                    <event-dispatching-element>
```


It's really `composed: false` that gives us new and interesting functionality.

When an event is dispatch with `composed: false` then that event will be contained within the shadow root in which it is fired. Right off, for the speed-obsessed developers reading this, that means your events will go faster! Whereas `{bubbles: false}` can double the speed of an event by completely cutting off the "bubble" phase (read half of the traveling required of an event), `{composed: false}` could cut that distance all the way down to two stops, the dispatching element and the shadow root that contains it, assuming such a simplified DOM tree. Code speed is likely not the concern here, but it is worth noting. What's really of most interest is access. When an event is dispatched with `composed: false` only the other elements encapsulated in its shadow root have access to it.

Yes, not only does shadow DOM allow you to encapsulate your CSS, DOM, and javascript, it will contain your events for you as well essentially making the element a closed application ecosystem. Within your sub-tree you could dispatch any number of events, with as simple (as affords the contained scope) or complex (as affords their lack of being public) event names as you'd like, process them as needed internally, and then only when needed or ready dispatch a new, clearly documented, and explicitly packaged event into the parent scope. `composed: false` is the [private fields](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Class_fields#Private_fields) of DOM events.

## The Responsibility Part

So, what are we to make of all this power? And, what sort of trouble can it get us in? After all, the premise behind such a broad assertion as "`composed: true` is harmful" is that it _will_, after a turn, get us in trouble.

My path towards examining this danger started with a conversation around the minutia that marks the difference between handing events via a passed callback and doing so via a listener. With a passed callback, you know that there is work that you need to do:

```js
const doWork = () => console.log('Do work.');
```

And you pass it into the element that needs to do that work.

```js
const primaryButton = ({onClick}) => html`
    <button @click=${onClick}>Primary Button</button>
`;

render(primaryButton({onClick: doWork}), document.body);
```

In this way you can pass this callback from a great distance if you need:

```js
const doWork = () => console.log('Do work.');

class PrimaryButton extend LitElement {
    static get properties() {
        return {
            onClick: { type: Function, attribute: false}
        };
    }
    render() {
        return html`
            <button @click=${this.onClick}>Primary Button</button>
        `;
    }
}

customElements.define('primary-button', PrimaryButton);

class Card extend LitElement {
    static get properties() {
        return {
            doWork: { type: Function, attribute: false}
        };
    }
    render() {
        return html`
            <div class="card">
                <h1>Something</h1>
                <p>Some stuff...</p>
                <primary-button .onClick=${this.doWork}></primary-button>
            </div>
        `;
    }
}

customElements.define('custom-card', Card);

class Section extend LitElement {
    static get properties() {
        return {
            doWork: { type: Function, attribute: false}
        };
    }
    render() {
        return html`
            <section>
                <custom-card .doWork=${this.doWork}></custom-card>
            </section>
        `;
    }
}

customElements.define('custom-section', section);

render(html`<custom-section .doWork=${doWork}></custom-section>`, document.body);
```

But, in the end, the work is done _AT_ the site of the event. In this way, even if you know work might need to be done high up in your application, you use a templating system (in the above example `lit-html` via `LitElement`) to pass that action down to the event site. This approach works perfectly with `composed: false` because with the callback passed into the `<primary-button>` element only the `<button>` element therein really needs to know about event that is being dispatched. However, we've just learned the `click` events (and other default UI-events) are generally dispatch with `composed: true`, so that means we _could_ also do the following:

```js
const doWork = () => console.log('Do work.');

class PrimaryButton extend LitElement {
    render() {
        return html`
            <button>Primary Button</button>
        `;
    }
}

customElements.define('primary-button', PrimaryButton);

class Card extend LitElement {
    render() {
        return html`
            <div class="card">
                <h1>Something</h1>
                <p>Some stuff...</p>
                <primary-button></primary-button>
            </div>
        `;
    }
}

customElements.define('custom-card', Card);

class Section extend LitElement {
    render() {
        return html`
            <section>
                <custom-card></custom-card>
            </section>
        `;
    }
}

customElements.define('custom-section', section);

render(html`<custom-section @click=${doWork}></custom-section>`, document.body);
```

In the above example, we _listen_ for the event, which is possible because the `click` event has `composed: true` by default. In theory, both samples of code output the same user experience, but that isn't true. While the passed callback example will ONLY call `doWork` when the `<button>` element in the `<primary-button>` element is clicked, the listening example will do so AS WELL AS calling `doWork` when any other part of the `<custom-section>` element is clicked: the `<p>`, the `<h1>`, the `<div>`, etc. Here is the source of "`composed: true` considered harmful". While the `composed: true` event allows you to listen more easily to the event in question, it also hears a lot more than you might be expecting when opting into the practice. Via the passed callback approach you could also go one step further with your callback, leverage the `stopPropagation()` method we discussed and prevent DOM elements that would naturally be later in the event lifecycle from hearing the event:

```js
const doWork = (e) => {
    e.stopPropagation();
    console.log('Do work.');
}
```

We're feeling safe now, aren't we!?

### Non-standard Events

A `click` event, and generally all `MouseEvents`, is pretty powerful in this way: they can happen everywhere. Without passing a callback, you would be forced to rely on [event delegation](https://davidwalsh.name/event-delegate) to contain the effects of such broadly felt/originated events. While this may seem powerful (and is leveraged in a very popular synthetic event system), it inherently breaks the encapsulation provided by the shadow DOM boundaries outlined by our custom elements. That is to say, if you _have_ to know that `<custom-section>` has a `<custom-card>` child that subsequently has a `<primary-button>` child that then has a `<button>` child, in order to respond to a click then why have encapsulation, to begin with? So, `composed: true` is harmful, after all? Well, there is one more thing to take into account, when we're talking about manually dispatch events, we get to decide what those events are called.

Our non-standard events whether they're made via `new Event('custom-name')` or `new CustomEvent('custom-name')` or `class CustomNamedEvent extends Event { constructor() { super('custom-name'); } }` are completely under our control. This means we no longer have to worry about the generic nature of the `click` event and can use a custom naming system to dispatch more specific (e.g. `importing-thing-you-care-about`) event names. By this approach, we get back a good amount of control over our response to an event:

```js
render(html`<custom-section @importing-thing-you-care-about=${doWork}></custom-section>`, document.body);
```

In this context, we can know that nothing but what we expect to dispatch the `importing-thing-you-care-about` event will be doing so. By this approach, we can listen from a distance, and be sure that only the element that we expect to dispatch the event is doing so, without having to resort to techniques like event delegation. Does it make your use of `composed: true` in this case safe? This starts to come down to the specific needs of your application.


## Recap

- DOM events are very powerful (even when only looking at the `bubbles`, `cancelable`, and `composed` settings as we have today) and can be leveraged for any number of things in an application.
  - `bubbles` controls whether the event enters the second half or "bubble" phase of its lifecycle
  - `cancelable` allows for `preventDefault()` to send an approval signal back to the dispatching element
  - `composed` decides how the event relates to shadow DOM boundaries
- If you've worked with these events before (whether in shadow DOM or not) you're likely accustomed to the way that almost all of them include `composed: true` by default.
- `composed: true` opens the event to being listened for at a distance, so the naming of that event becomes more important.
- When passing a callback into a component for an event, `composed: false` can give fine-grained control over an application's ability to react to that event.

# `composed: true` considered harmful?

With all this new knowledge, what do you think, should `composed: true` be considered harmful? Is the browser killing us with a thousand cuts by setting all UA-dispatched UI events to `composed: true` by default? I've used both values of `composed` in my own manually dispatched events and it's hard to say one is specifically better/more dangerous than the other. I'm pretty sure, like most technical decisions, that what value you use for `composed` should be decided based on the specific needs of your application and/or the offending component in question. However, my experience is just that, my experience, I'd love to hear about yours! Please hop into the comments below and share whether you've been harmed by `composed: true` and how.

## Want to do more research?

Still wrapping your brain around what all this looks like? I've put together an event playground where you can test the various settings and realities we've discussed so far: 

{% glitch super-area app %}

While the design therein could certainly be considered _harmful_, hopefully, it'll give you a more clear understanding of the settings that can be applied to events and how that affects the way those events travel around the DOM. Take note that each DOM element that hears an event will say so, along with the phase during which it heard the event, what step in the path of the event it passed through that element and the `target` element at that point next to the original dispatching element. I use manually dispatched events pretty liberally across my applications and shadow DOM-based components, and putting this little ditty together went a long way to cementing my knowledge of DOM events, so hopefully, it helps you too. As you get deeper into your studies, if you remix the project to help outline your thoughts on `composed: true`, please share them with us all in the comments below. 

First of all, what even is `composed: true`, and when _might_ you use it?

Native HTML elements communicate up the DOM tree using DOM events. You might be used to seeing this with elements like `<input />` which publish events like `change` and `input`, or with the `<button />` element, where it's common to rely on the `click` event that it publishes. It might not be immediately clear you are relying on these things, but if you've worked with `onclick` (native) or `onChange` (virtual DOM) properties, it is these DOM events that you are relying on under the hood.

For a good long time, developers have also been able to rely on this pattern for communicating along the DOM tree via the [`el.dispatchEvent()`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/dispatchEvent) method which accepts any of the many native events available in the browser (e.g. `new Event()`, `new MouseEvent()`, `new InputEvent()`, etc.) or an event of your own creation via `new CustomEvent()`. These event constructors generally accept two arguments; `typeArg`, a DOM string with the name of the event and `eventInit`, an optional dictionary of properties for more finely defining the event you are creating.

The `eventInit` argument gives a developer many important options to work with when creating an event. Two particularly powerful properties of this argument are `bubbles` and `composed`. `eventInit.bubbles` outlines whether the event in question will bubble (or travel) up the DOM from the element on which it was dispatched, while `eventInit.composed` (the subject of today's discussion) is used to determine whether the event exists only in the local document fragment or across the entire document.

## Say what?

So, for example, a [click event](https://developer.mozilla.org/en-US/docs/Web/API/Element/click_event) in a simple HTML document will originate at the document boundary (`document`), capture down the DOM tree before arriving at the dispatching element (often an `a` or `button`), and then subsequently bubbling back up the DOM tree until returning to the `document`. This is due to their being no `document-fragments` to worry about and the native `click` event having `bubbles` set to `true` by default. Bubbling events acting in this way is something that developers have come to expect from the web in that our pages have been single document realities since the beginning of the DOM. However, with the advent of [Shadow DOM](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_shadow_DOM), and our ability to add it to both [Custom Elements](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements), as well as many of the DOM elements we've worked with since well before the [Web Component](https://developer.mozilla.org/en-US/docs/Web/Web_Components) specifications came into reality, we now have the possibility of a `document` that consists of many `document-fragments`.

When a click event is dispatched inside of a `document-fragment` as provided by a `shadowRoot` then the capture phase of that event will start from the `document-fragment` unless `composed` is set to `true`. Equally, the bubble phase (when `bubbles` is set to `true`) will end at that same `document-fragment` while the value of `composed` is not `true`. However, due to the fact that a `shadowRoot` is meant to encapsulate the DOM and scripting that goes on within it, when an event has a `composed` value of true, the capture and bubble phases will list the `target` of the event as the closest of the dispatching element or and element with a `shadowRoot` attached. That is to say, regarding the following HTML, that `click` events on the `<a>` will register a target of `<child-with-shadow-root>` when listened for on the `<parent-with-shadow-root>` element:

```html
<parent-el> <!-- listeners here get `child-with-shadow-root` as the target -->
    #shadow-root
        <child-with-shadow-root>
            #shadow-root
                <a>Click here!</a> <!-- click happens here -->
        </child-with-shadow-root>
</parent-el>
```

Like so many things in HTML and JS, this can be overcome by querying the `event.composedPath()` method for an array of all the elements where you'll find the zeroeth element to be the true dispatcher of the event.

## So, why is it considered harmful?

One reason is what we've just learned about how it is possible to work around the encapsulation. However, knowing an event might occur, listening for it, and then querying `event.composedPath()` is a non-trivial amount of work to defeat said encapsulation, especially when a _really_ interested party can use `el.shadowRoot.querySelector('...')` to get into your element from the front door (baring `el.attachShadow({mode: 'open'});`, of course). For this, we can probably give `composed: true` a break, though please let me know your thoughts, this is a `#discuss` article after all.

A slightly more pressing, and somewhat vexing, reason came up recently when I was discussing the difference between the "pass callbacks down" pattern so prevalent in virtual DOM contexts and the "listen for events" pattern that is more natural when working with actual DOM. From my experience, I had approached these patterns as one requiring a fixed dependency (if the parent didn't pass something to do nothing got done) vs a self-resolving dependency (if something needed to happen, it would at some point). In said discussion, the above sentence was remixed in a way I previously hadn't given much thought, one of these patterns is a direct dependency (you know exactly where it came from and why it is used) vs a spooky action at a distance (something happened, somewhere, and it's unclear whether you should do something about it). "Spooky action at a distance" is a pretty bad word in data management, so when I _really_ heard this I started wondering, _should `composed: true` be considered harmful?_

## And, if I _do_ use events?

https://glitch.com/edit/#!/super-area?path=script.js:11:17

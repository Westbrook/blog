---
layout: layouts/post.html
title: I'm feeling so disconnected
date: 2025-02-05
modified: 2025-02-05
---

Have you ever been nerd sniped?

[![Nerd Sniping](/patterns/nerd_sniping.png)](https://xkcd.com/356/)

Ok, maybe this is more of a call-out. And, it starts like this:

> [@westbrook](https://mastodon.social/@westbrook) [@nachtfunke](https://indieweb.social/@nachtfunke) holy shit please make this a blog post — so much great info packed into a tiny toot
> 
> — [@zachleat](https://fediverse.zachleat.com/@zachleat) ([link](https://fediverse.zachleat.com/@zachleat/113951721381011053))

Zach is a purveyor of fine things like [11ty](https://www.11ty.dev/), which powers this blog, so when he asks someone to blog something...you listen. Also, as a person who "asks" people to blog about their learnings, I try to _be an example_. 🫣

So, here we are.

---

It..._started with a toot_.

> [@nachtfunke](https://indieweb.social/@nachtfunke) In general, if you have work that belongs in `connectedCallback()` then you either need to clean it up in `disconnectedCallback()` (which means it can happen again with no issues), move to the future and use `moveBefore()` [https://caniuse.com/mdn-api_element_movebefore](https://caniuse.com/mdn-api_element_movebefore) (no disconnection), leverage `<slot>`s and shadow DOM (no disconnection), or _know_ that you are going to be wrapping the element in the same tree and gate the `connectedCallback()` work on that knowledge (managed disconnection).
>
> — [@westbrook](https://mastodon.social/@westbrook) ([link](https://mastodon.social/@westbrook/113951513169639844))

Hop into the thread, if you'd like, and get a load of what Thomas was running into, people's thoughts about it, and how he solved it (hopefully he'll share when he does). The pertinent info is that he was working [Custom Elements](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements) and when using a parent to wrap children those children were firing their [`connectedCallback()`](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements#lifecycle_callbacks) more often than he expected/wanted/hoped. As an author/consumer/chairperson of the community group of web components for several years now, I've got some patterns that I use when trying to do something like this:
- clean up work done in `connectedCallback()` in the matching `disconnectedCallback()` lifecycle method
- wrap content with `<slot>`s in a shadow root to apply an updated DOM tree
- gate work done in `connectedCallback()` on lifecycle realities specific to your custom element
- fast forward _to the future_ and leverage [`moveBefore()`](https://caniuse.com/mdn-api_element_movebefore) _(editors note: this _not_ being the future, I've not actually used this myself 🙈)_

What are they and how do they work?

## Get a life(cycle)

Once you've registered a custom element:

```js
customElements.define('my-el', class extends HTMLElement {});
```

You have the _option_ to include various lifecycle methods in the class describing your custom element. Above, you'll see that none are included. Below, you'll see a sampling of them put to completely non-productive use.

```js
customElements.define('my-el', class extends HTMLElement {
    constructor() {
        super();
        console.log('I have been constructed');
 	}
    connectedCallback() {
        console.log('I have been connected!');
 	}
    disconnectedCallback() {
        console.log('But, I liked being connected 🥺');
	}
});
```

Useful or not, some tasks really are _one-time_. When you've got tasks like that, put them in a one-time lifecycle method; `constructor()`. Described in this way a custom element will never be able to do that work again! If that's your use case, feel free to take the rest of the day off.

Productivity is in the eye of the application, but no matter what you might do in a `connectedCallback()` was it to have a persistent effect on the world around it, it is highly recommended that you _undo_ those effects in your `disconnectedCallback()`. A great example of that is adding and removing event listeners that listen to things outside of your custom element. For example, if you listen to the `resize` event on `window`, something I did just last week.

```js
customElements.define('my-el', class extends HTMLElement {
    handleWindowResize = () => {
        console.log('Yes, I _have_ changed size. And, I AM the window.');
 }
    connectedCallback() {
        /**
 		 * I am an event dispatched from DOM _outside_ of the custom element that responds to this event.
 		 * As long as this binding is active, the listener will be called, and because the listener
 		 * can be called the custom element in question can never be garbage collected. 😱
 		 **/
        window.addEventListener('resize', this.handleWindowResize);
 	}
    disconnectedCallback() {
        /**
 		 * Now we're getting so fresh and so clean, cleaning up after ourselves. 🧼
 		 **/
        window.removeEventListener('resize', this.handleWindowResize);
 	}
});
```

Not everything that you might do in a `connectedCallback()` can be taken back, or at least not in a one-for-one manner. So, while this technique should always be top of mind, it won't solve all your problems. Luckily, it's not the only tool in the belt!

## ~~Stop~~ Start projecting

Shadow DOM gets a bad rap. "You can't style it like other DOM!" ...it wasn't _meant_ to be styled like other DOM. "You can't make it accessible!" ...you can, just not exactly how you make _other_ DOM accessible. AND, new, even more flexible, APIs for this [are landing soon™️](https://github.com/WebKit/WebKit/pull/39742). "You can't progressively enhance it!" ...actually, [you can](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM#declaratively_with_html) 🤯.
People go on and on, so I won't. I'll focus on one of the best features of shadow DOM, the `<slot>` element.

A `<slot>` is like a movie screen inside of your shadow DOM onto which you can _project_ content from the outside. Move the `<slot>` and the projection moves with it. Remove the `<slot>` and the projection goes dark. Rename the `<slot>`, readdress the content to a different `[slot]`, etc., etc., hopefully, you're getting the point. However, because all of this happens via _projection_, the content being projected onto your `<slot>` will never be connected or disconnected when changing where or if the projection is available.

This means that if you first write your shadow root as:

```html
<parent-element>
    <template shadowrootmode="open">
        <slot></slot>
    </template>
    <child-element></child-element>
</parent-element>
```

And then later, update your shadow root as follows, the `<child-element>` will _appear_ as it is in a new location without needing to be disconnected and connected again.

```js
document.querySelector('parent-element').shadowRoot =
`<ul>
	<li><slot></slot></li>
</ul>`;
```

_(Editors note: just because you can edit an open shadow root like this doesn't mean you should. This example is meant to be illustrative NOT normative.)_

Without being exhaustive of all the ways this approach can come into play, I will acknowledge that while the example above works great for _one_ `<child-element>`, you're likely to want a new `<li>` wrapper for the `<slot>` onto which you project _each_ `<child-element>` that a `<parent-element>` contains. How does one do that without altering the light DOM? Well, [imperative `<slot>` assignment](https://developer.mozilla.org/en-US/docs/Web/API/HTMLSlotElement/assign) can help. The leased nuanced version of which might look like:

```js
customElements.define('parent-element', class extends HTMLElement {
  	constructor() {
        super();
        // One time work done in a constructor... wasn't someone just talking about that? 😲
        this.attachShadow({mode: 'open', slotAssignment: "manual"});
        this.updateShadowDOM();
 	}
  	updateShadowDOM = () => {
    	this.shadowRoot.innerHTML =
`<ul>
${[...this.children].map((_,i) => `<li><slot name="child-${i}"></slot></li>`).join('')}
</ul>`;
    	this.assignChildren();
 	}
  	assignChildren() {
    	this.shadowRoot.querySelectorAll('slot').forEach((slot, i) => {
      		slot.assign(this.children[i]);
 		});
 	}
});
```

Give it a test drive on [CodePen](https://codepen.io/Westbrook/pen/ZYzdevP). Change it. Make it better. _Finish it._ And then, share it with me on [Mastodon](https://mastodon.social/@westbrook)!

No, this doesn't manage late-added DOM. Add a [mutation observer](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver), likely via the `connectedCallback()` and `disconnectedCallback()` lifecycle methods 😉.

No, this doesn't address styling.

Yes, it is accessible. When you use native DOM elements to outline the content structure, this comes for free 🦋.

No, it doesn't take into account progressive enhancement. Not everything can or should be progressively enhanced. Maybe you know a way to progressively enhance this? Return to step "share it back to me on Mastodon" and then the world will know it too!

## Pay the toll

New York City and the MTA just instituted a [Congestion Relief Zone](https://congestionreliefzone.mta.info/tolling) to reduce traffic in midtown Manhattan. It's just been a few weeks and it's already been great. _Insert photo of pristine streets full of pedestrians._ If your `connectedCallback()` is also getting too much traffic, you should consider instituting one yourself. Maybe it looks like this:

```js
customElements.define('child-element', class extends HTMLElement {
    dontMindMe = false;
    conectedCallback() {
        if (this.dontMindMe) {
            return;
 		}
        // Do expensive, side-effectful, magic work that you don't want happening all the time.
 	}
});
```

This could be a _well-known_ API (that means to document this, no one reads that documentation BUT you can point to it when they have trouble) and it can be leveraged like:

```js
const child = document.querySelector('child-element');
child.dontMindMe = true;
child.remove();
// other work changing the connected state of the element
document.body.append(child);
child.dontMindMe = false;
```

This puts responsibility onto this consumer to know when their work should be side effectful and open/close the gate accordingly. This is my favorite sort of responsibility, the kind that comes in the form of a superpower, but depending on your context and the scope of actions that need to be managed, you might like to take all the power for yourself.

To do so, this sort of gating can also be handled on various purpose-built methods:

```js
customElements.define('child-element', class extends HTMLElement {
    moveTo(target) {
        this.dontMindMe = true;
        target.append(this);
        this.dontMindMe = false;
 	}
    dontMindMe = false;
    conectedCallback() {
        if (this.dontMindMe) {
            return;
 		}
        // Do expensive, side-effectful, magic work that you don't want happening all the time.
 	}
});
```

Which would allow a consumer to do the following:

```js
const child = document.querySelector('child-element');
const newParent = docuement.getElementByIt('new-parent');
child.moveTo(newParent);
```

Slightly less responsibility on the consumer and much more power to you. You know what they say about more power, however. On the other side of _this_ gate is road to managing every sort of DOM interaction that can be played out against your custom element, so be wary!

## The future?

So, I made this grand statement about `.moveBefore()` before using the API. Naively, I'd want it to [work like this](https://codepen.io/Westbrook/pen/ogvrWgZ).

Look at how many times this code logs "I have been connected!"...

```html
<div id="target">Target</div>
<hr>
<hr>
<my-el>Element</my-el>
<script>
customElements.define('my-el', class extends HTMLElement {
    constructor() {
        super();
        console.log('I have been constructed');
 	}
    connectedCallback() {
        console.log('I have been connected!');
 	}
    disconnectedCallback() {
        console.log('But, I liked being connected 🥺');
 	}
});

target.moveBefore(document.querySelector('my-el'), null);
</script>
```

Two times. That's too many times. Was I wrong?

Was the connected state not part of the state that `.moveBefore()` was meant to preserve? 😳

Or, was I just uneducated?

Here's where it is good to "read the [spec](https://github.com/whatwg/dom/pull/1307/files#diff-30f698d9b4650514daf7a04de5e44eebf8feff1a30e0505f8eeaf06703b2777dR2990-R2992)". Even if the "spec" is still just an open PR. Along with adding the `.moveBefore()` method, this new API also adds the `connectedMoveCallback()` lifecycle method to a custom element. By default, it doesn't do anything new. BUT, if you [include it in your custom element definition](https://codepen.io/Westbrook/pen/RNbzVWx)...

```html
<div id="target">Target</div>
<hr>
<hr>
<my-el>Element</my-el>
<script>
customElements.define('my-el', class extends HTMLElement {
    constructor() {
        super();
        console.log('I have been constructed');
 	}
  	connectedMoveCallback() {
    	console.log('I will move before you.');
	}
    connectedCallback() {
        console.log('I have been connected!');
	}
    disconnectedCallback() {
        console.log('But, I liked being connected 🥺');
	}
});

target.moveBefore(document.querySelector('my-el'), null);
</script>
```

Then "I have been connected!" _only_ gets logged once, and we have a new log of "I will move before you.", just as Berners-Lee intended.

This is opting-into the `.moveBefore()` behavior without accidentally effecting exiting custom element definitions without their knowledge.

## Hammers and nails

Will any one of the above tools solve all of your problems when managing DOM interactions made on your custom elements?

No.

Do we now have a lot more tools in the belt with which to address such situations?

Hopefully.

Are there other tools you use that I haven't thought of here?

Then it's up to you now. Make an 11ty blog, toot about your technique, get called out to blog about it, and don't back down in the face of sharing your technique(s) with the world!


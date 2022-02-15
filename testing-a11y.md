Recently, I had the opportunity to discuss the difficulties, learnings, and victories or developing [Spectrum Web Components](https://opensource.adobe.com/spectrum-web-components/) together with fellow custom element developers from teams at [IBM](https://web-components.carbondesignsystem.com/?path=/story/introduction-welcome--page), [ING](https://lion-web.netlify.app/), [SAP](https://sap.github.io/ui5-webcomponents/), and [Vaadin](https://vaadin.com/components). If you missed the live stream, [check out the recording](https://www.youtube.com/watch?v=xz8yRVJMP2k&feature=youtu.be)! Fellow panelist, Ari Gilmore, made a great point that there is a lack of reading material for developers like ourselves to draw from when looking to build solid accessibility practices into the web components space. With that in mind, I thought it would be a good idea to take some of the abstract concepts we discussed in the panel and share some actual examples of working and testable code. Hopefully, this can better support the next developer(s) looking to bring a high-quality, accessible, design system to life for their team via web components.

To support this conversation, I'll be bringing to life an input element pattern featuring accessible labeling and help text. Taking the suggestion of Thomas Allmer and the team at ING, the first example will be a no shadow DOM implementation with associated testing. With a shared baseline on how both the HTML and the testing works, we'll explore some different examples of delivering the relationship an input element, label element, and help text element accessibly with custom elements and shadow DOM. We'll talk about ways that we can mix and match these approaches and how some of the approaches align with or support various in-development and draft specifications for making this process even less work.

Some particularly prescient subjects that we went over during the panel that I'll dive into in this article:
- leveraging the [axe-core](https://github.com/dequelabs/axe-core) accessibility testing engine
- seeing and understanding the accessibility tree
- using native keyboard interactions at testing time
- how ID references do not pass through a shadow boundary
- ways to mimic native element references

I'll also connect some dots between these techniques and the various web component libraries my fellow panelists have brought into the world that leverages them so you can follow up on what is needed to take those patterns into production.

Some subjects that I won't spend much time on in this article, but are of great importance when shipping high quality, production-ready implementations of these patterns, include:
- content styling
- form association
- state management
- validation

Each of these, and likely other topics omitted without reference, would easily fill their own article(s), and hopefully, the support you'll find here in getting a jump on making your shadow DOM-based content more accessible will free you up to share your approach to these realities, next.

---

# Table of contents

- [Starting from HTML](#starting-from-html)
- [What and how to test](#what-and-how-to-test)
  - [axe-core](#axecore)
  - [Accessibility tree](#accessibility-tree)
  - [Native keyboard events at test time](#native-keyboard-events-at-test-time)
- [How should we build it?](#how-should-we-build-it)
  - [Factoring from raw HTML](#factoring-from-raw-html)
  - [Wrapper](#wrapper)
  - [Decorator](#decorator)
  - [Emitter](#emitter)
      - [The Decorator Pattern Plus](#the-decorator-pattern-plus)
      - [Panelist projects leveraging this technique](#panelist-projects-leveraging-this-technique-1)
  - [Outside-in](#outsidein)
      - [ID references DO NOT pass through shadow boundaries](#id-references-do-not-pass-through-shadow-boundaries)
      - [Panelist projects leveraging this technique](#panelist-projects-leveraging-this-technique-2)
  - [Snowflakes](#snowflakes)
      - [Pretending to be a native element](#pretending-to-be-a-native-element)
          - [Responding to the "for" attribute](#responding-to-the-for-attribute)
          - [Observing text content](#observing-text-content)
      - [Panelist projects leveraging this technique](#panelist-projects-leveraging-this-technique-3)
- [In memoriam](#in-memoriam)
- [In the next life...](#in-the-next-life)

---

**Disclaimer**
Before we get started, as I mentioned in the panel, I wouldn't call myself an accessibility specialist. I understand accessibility to be an important part of delivering products to people and strive for the tools I leverage to do so to be more and more accessible over time. In the community, I work with smart, caring people, like those that I joined on the panel, to find new and better ways to do things. At Adobe, while developing Spectrum Web Components, I work with a dedicated team of accessibility engineers, some of whom actually write the specs to which the entire web community develops software. Without their patience and support, I'd definitely not have gotten as far as I have in being able to bring accessible surfaces to the web. That certainly doesn't mean I get everything right. So, while I hope you find this article similarly useful, the only way for us all to get a little more accessible is for you to share in the comments in you find something I've missed, or a different way to achieve the same goals, or want to know something beyond what I'll be coving. We can all make the web a little better place, together!

# Starting from HTML

Following the lead of the ING team who's solid work has brought up [Lion](https://lion-web.netlify.app/), we'll start from the raw HTML pattern that delivers a labelled and described `<input>` element:

```html
<div>
    <label for="input">Label</label>
    <input id="input" aria-describedby="description" />
    <div id="description">Description</div>
</div>
```

You can [view a demo of this code](https://webcomponents.dev/edit/9XdUWIy1wjJqyLFyii0p/stories/index.stories.js?p=stories) or [clone it from GitHub](https://github.com/Westbrook/testing-a11y) to look more closely into how it works. However, the crux of the functionality (all provided natively by HTML at this point) is ID reference. 

Our `<label>` element accepts the `for` attribute which references an element by ID. The element it references, in this case, our `<input>` elements, will be the one that receives the content of the `<label>` as its "name" in the accessibility tree passed the screen reader. Locally, it will also pass the `click` and `focus` events triggered by clicking the `<label>` onto the referenced element as well. This is less important in an `<input type="text">` element, however types like `checkbox` or `radio` will be toggled as appropriate when passing these events which can make both interactive with and styling your form content easier.

The `<input>` element in question is leveraging the `aria-describedby` attribute, which also references an element by ID. Here, the attribute points to our `<div>` element which holds our description. There is no default interactive functionality that this relationship provides, but it will supply the text content of the referenced element as the "description" of the `<input>` element in the accessibility tree.

# What and how to test

All of this is a great start to delivering this pattern accessibly, but don't just take my word for it. Let's dig into how we can test these things to be true, and at the same time set the table for refactoring this pattern to leverage web component APIs like custom elements and shadow DOM.

## axe-core
> Thanks, @open-wc/testing!

The first stop in any accessibility testing (or possibly any UI testing) should be on tests you can get "for free". For our use case, that can be provided by the [axe-core](https://github.com/dequelabs/axe-core) accessibility testing engine as packaged into [Chai a11y Axe](https://open-wc.org/docs/testing/chai-a11y-axe/) and delivered in [`@open-wc/testing`](https://www.npmjs.com/package/@open-wc/testing). While you may have caught me quoting a woefully small percentage of issues that automated testing like this can catch, I am heartened to hear that Deque (makers of axe-core) have recently looked deeper into this situation and believe that [57.38% of accessibility issues can be discovered via automation](https://accessibility.deque.com/hubfs/Accessibility-Coverage-Report.pdf), so this will be a big first step in affirming the accessibility of the patterns you deliver.

What's more, this big first step is actually a really small step in reality. Check out the code below for confirming the accessibility of a DOM fixture with axe-core via Chai a11y axe:

```js
// Has the side effect of binding Chai A11y aXe to `expect`
import { fixture, expect } from '@open-wc/testing';

// ...

it('passes the aXe-core audit', async () => {
    const el = await fixture<HTMLDivElement>(`
<div>
    <label for="input">Label</label>
    <input id="input" aria-describedby="description" />
    <div id="description">Description</div>
</div>
`);

    // Asynchronously tests the accessibility of the supplied content
    await expect(el).to.be.accessible();
});
```

That's right, the guts of the test at `await expect(el).to.be.accessible();` and you'll immediately start getting reports of the accessibility achieved by the DOM in your fixture. Visit the [Rule Descriptions](https://github.com/dequelabs/axe-core/blob/develop/doc/rule-descriptions.md) outlining all of the concepts that will be covered simply by adding the one test.

This one test is so important that many tools get out in front of you to ensure it is included from the start. `npm init @open-wc` will include this test by default when `testing` is added by the generator. At Spectrum Web Components, when generating a new package with our [Plop templating](https://github.com/adobe/spectrum-web-components/blob/main/projects/templates/plop-templates/test.ts.hbs#L19-L29), we have this test by default as well. However you are initializing your projects, I highly suggest you work to have this sort of test included by default.

With that out of the way, and as many as 57.38% of your accessibility issues already caught, we can take a look at some more nuanced contexts that can be useful to have covered.

## Accessibility tree
> Thanks, @web/test-runner!

The DOM tree, paired with various ID references and `aria-*` attributes is used by the browser to construct an accessibility tree that it represented to screen readers to support visitors experiencing and interacting with our content with their assistance. Leveraging [WAI-ARIA Authoring Practices](https://www.w3.org/TR/wai-aria-practices-1.2/), along with the assurances that axe-core can set as a foundation, we can generally be sure of what the tree a browser might build from our content. However, it can beneficial to know for sure.

One path to this is to leverage [full accessibility tree](https://developer.chrome.com/blog/full-accessibility-tree/) view in Chrome DevTools. Switch this on and you can manually confirm the tree to which your code is being converted by the browser. Many browsers surface the ability to see _part_ of the accessibility tree via their developer tooling, which is very useful, but you often end up needing to rely on manual testing to confirm that the right content is being delivered to screen readers. 

While manual testing should definitely be a part of your go to production strategy, knowing what's actually in that tree, not just the relations that get created but the actual content that is bound by those relationships, can be an important addition to our automated testing workflows. To support this, browser runners like [Playwright](https://playwright.dev/docs/api/class-accessibility) surface APIs by which we can access the accessibility tree (as a snapshot thereof) directly.

[`@web/test-runner`](https://www.npmjs.com/package/@web/test-runner) give you access to these APIs while unit testing via its [commands](https://modern-web.dev/docs/test-runner/commands/#accessibility-snapshot) interface. This allows you to snapshot the accessibility tree at any point during an interaction with your code at test time, and confirm that nodes and relations that you expect to exist are actually there.

```js
import { a11ySnapshot, findAccessibilityNode } from '@web/test-runner-commands';

// ...

const label = 'Label';
const description = 'Description';

it(`is labelled "${label}" and described as "${description}"`, async () => {
    const el = await fixture<HTMLDivElement>(`
<div>
    <label for="input">${label}</label>
    <input id="input" aria-describedby="description" />
    <div id="description">${description}</div>
</div>
`);

    const snapshot = (await a11ySnapshot({})) as unknown as DescribedNode & {
        children: DescribedNode[];
    };

    const describedNode = findAccessibilityNode(
        snapshot,
        (node) =>
            node.name === label &&
            node.description === description
    );

    expect(describedNode, `node not in: ${JSON.stringify(snapshot, null, '  ')}`).to.not.be.null;
});
```

The above code actually builds our HTML implementation of the labeled and described input, appends it to the document, and then requests a snapshot of the accessibility tree. Then, using `findAccessibilityNode` helper confirms the presence of a node meeting the requirements supplied. You'll also note that I leverage the custom error message of the Chai expectation here to allow for seeing the stringified accessibility tree when a test fails. As the tree can have many edge cases, and unexpected results, I've found this an important part of understanding what I'm testing.

One way that the accessibility tree surprises me, in the case of this example, is that WebKit does not actively associate the description content with our `<input>` element. While manual testing does confirm that the description text is associated appropriately, the tree will not return a reality where these two elements are connected. With the cross-context manual testing to back up this relationship in a WebKit browser, I'm comfortable with expanding the test in this case to something that takes this deviation into account:

```js
export const findDescribedNode = async (
    name: string,
    description: string
): Promise<void> => {
    await nextFrame();

    const isWebKit =
        /AppleWebKit/.test(window.navigator.userAgent) &&
        !/Chrome/.test(window.navigator.userAgent);

    const snapshot = (await a11ySnapshot({})) as unknown as DescribedNode & {
        children: DescribedNode[];
    };

    // WebKit doesn't currently associate the `aria-describedby` element to the attribute
    // host in the accessibility tree. Give it an escape hatch for now.
    const describedNode = findAccessibilityNode(
        snapshot,
        (node) =>
            node.name === name && (node.description === description || isWebKit)
    );

    expect(describedNode, `node not in: ${JSON.stringify(snapshot, null, '  ')}`).to.not.be.null;

    if (isWebKit) {
        // Retest WebKit without the escape hatch, expecting it to fail.
        // This way we get notified when the results are as expected, again.
        const iOSNode = findAccessibilityNode(
            snapshot,
            (node) => node.name === name && node.description === description
        );
        expect(iOSNode).to.be.null;
    }
};
```

You'll see that this test uses the user agent to compare for WebKit and then allows for the test to pass when the description isn't related to the `<input>` and the browser is WebKit. To support understanding when/if this reality were to change in the future, the test then runs the expectation in reverse for WebKit so that a failure would be raised in our test and the workaround can be removed.

In other testing scenarios, WebKit will associate description content without issue, so pay attention to your results and pair them with manual testing when setting the baselines from which you want to protect against regression. I've also seen these sorts of differences in cross-browser understandings of the `role` of certain patterns, so be aware of the tree your content is creating when deciding the appropriate testing pattern to apply.

## Native keyboard events at test time
> Thanks, Playwright!

Once you are comfortable with the experience you are delivering to screen reader users, another user segment to ensure you are accessibly supporting keyboard navigation users. Whether to guide the screen reader across your content, or in support of other situations, it is important to know that your content can be accessed via the keyboard in expected ways. This can often prove elusive as testing keyboard events at unit test time is much more complex than it may seem. However, once you've established a reliable path to do so, the techniques leveraged to test this can be useful in other areas, as well; for instance, in UIs that include things like `<input>` element, which would traditionally be interacted with via the keyboard by all users.

Synthetic keyboard events can provide you with a decent entry into this area:

```js
const keyboardEvent = (
    code: string,
    eventDetails = {},
    eventName = 'keydown'
): KeyboardEvent => {
    return new KeyboardEvent(eventName, {
        ...eventDetails,
        bubbles: true,
        composed: true,
        cancelable: true,
        code,
        key: code,
    });
};
```

Especially when the feature at test is also fully synthetic, e.g. something that you've _added_ to your input element, they can take you a long way. When you're looking to test more complex keyboard interactions, including the various phases on a keypress, you might even turn to a testing library or framework to manage the complexities therein with success.

Once you move beyond testing your code directly, and into how your code should work in concert with the browser in which it is delivered synthetic events begin to show their shortcomings. This can be seen when attempting to use a synthetic event to alter the value of an `<input>` element. To fully mimic these interactions, more and more synthetic events need to be stacked on top of imperative commands to the to `<input>` until the whole process becomes quite brittle. Go one extra step and make expectations when a `Tab` key is pressed, and the approach falls apart altogether.

A native keyboard event doesn't just have all phases of a keypress that can be difficult to synthesize reliably, the browser itself recognizes the native keyboard interaction and responds with functionality beyond that which is found in your code. That means that the event has to originate with the browser itself. This is where tools like [Playwright](https://playwright.dev/docs/api/class-keyboard) can step in and give you support for native keyboard interactions. Again, [`@web/test-runner`](https://www.npmjs.com/package/@web/test-runner) gives you access to these APIs while unit testing via its [commands](https://modern-web.dev/docs/test-runner/commands/#send-keys) interface. Leveraging this allows us to place `<input>` elements before and after the code we have under test and ensure that `Tab` and `Shift + Tab` interactions behave as expected. Code for doing so might look as follows:

```js
import { sendKeys } from '@web/test-runner-commands';

// ...

it('is part of the tab order', async () => {
    const el = await fixture<HTMLDivElement>(`
<div>
    <label for="input">${label}</label>
    <input id="input" aria-describedby="description" />
    <div id="description">${description}</div>
</div>
`);
    const input = el.querySelector('input') as HTMLInputElement;
    const beforeInput = document.createElement('input');
    const afterInput = document.createElement('input');
    el.insertAdjacentElement('beforebegin', beforeInput);
    el.insertAdjacentElement('afterend', afterInput);
    beforeInput.focus();
    expect(document.activeElement === beforeInput, `activeElement: ${document.activeElement}`).to.be.true;
    await sendKeys({
      press: 'Tab',
    });
    expect(document.activeElement === input, `activeElement: ${document.activeElement}`).to.be.true;
    await sendKeys({
      press: 'Tab',
    });
    expect(document.activeElement === afterInput, `activeElement: ${document.activeElement}`).to.be.true;
    await sendKeys({
      press: 'Shift+Tab',
    });
    expect(document.activeElement === input, `activeElement: ${document.activeElement}`).to.be.true;
    await sendKeys({
      press: 'Shift+Tab',
    });
    expect(document.activeElement === beforeInput, `activeElement: ${document.activeElement}`).to.be.true;
    beforeInput.remove();
    afterInput.remove();
});
```
You'll see here that our test consists of three `<input>` elements and applies focus to the first before using `Tab` and `Shift + Tab` keyboard events to navigate through them. This may feel like testing code that isn't yours, and you might be right in this case of all native `<input>` elements in the same DOM tree. However, when shadow DOM boundaries come into play, it becomes more important to confirm how a keyboard user might come into contact with the elements you are building. 

# How should we build it?

There are a multitude of ways that we could structure this input experience with custom elements and shadow DOM. On top of each of those options is the ability to mix and match them across various contexts to make them work "just right" for your library or product. From here, let's dive into doing just that, while looking at some more "pure" implementations of the [Wrapper](#wrapper), [Decorator](#decorator), [Emitter](#emitter), [Outside-in](#outside-in), and [Snowflakes](#snowflakes) techniques, as well as how the some of the panelist projects leverage them, or combinations thereof.

## Factoring from raw HTML

We've already seen the raw HTML that we'll be working from, but here it is again as a reminder:

```html
<div>
    <label for="input">Label</label>
    <input id="input" aria-describedby="description" />
    <div id="description">Description</div>
</div>
```

See the [demo](https://webcomponents.dev/edit/9XdUWIy1wjJqyLFyii0p/stories/index.stories.js?p=stories) of [webcomponents.dev](https://webcomponents.dev).

[Clone the code](https://github.com/Westbrook/testing-a11y) on GitHub. 

Below, I've included five different ways to factor this raw HTML into custom elements, but they're just a small selection of the ways that you could do so accessibly. For each, we'll take a look at the custom elements that need to be created to leverage them, how those custom elements alter our usage in HTML, and what types of changes or additions might be needed to our test suite to support these decisions. I'll also link to examples of all or part of these techniques in work from my fellow panel members in the work on [Carbon Web Components](https://web-components.carbondesignsystem.com/?path=/story/introduction-welcome--page), [Lion](https://lion-web.netlify.app/), [SAP](https://sap.github.io/ui5-webcomponents/), and [Vaadin](https://vaadin.com/components), or in my own at [Spectrum Web Components](https://opensource.adobe.com/spectrum-web-components/).

## Wrapper

See the [demo](https://webcomponents.dev/edit/9XdUWIy1wjJqyLFyii0p/src/index.ts?branch=wrapper%40wfVpXdsTYhf9mY2slxW4t2Y2SjC3&p=stories) on webcomponents.dev.

[Clone the code](https://github.com/westbrook/testing-a11y/tree/wrapper) on GitHub.

This techniques is called "wrapper" because really, all that we're doing is wrapping our previously accessibly HTML with a custom element:

```html
<testing-a11y>
    <label for="input">Label</label>
    <input id="input" aria-describedby="description" />
    <div id="description">Description</div>
</testing-a11y>
```

That's it, you're done. Ship it!

This `<testing-a11y>` element relies on the native accessibility of the raw HTML that we started with and then encapsulates the reusable functionality that you'd actually want to ship in a custom input element within the parent element wrapper. By itself, however, it places a lot of responsibility on a consuming developer to ensure that each usage fully completes the contract of accessibility promised by the raw HTML from which we started. I'd guess that this higher level requirement in their consumers lead other members on the panel not to use this technique as well, but you can always reach out to them and their teams for more information.

In the case that you like the flexibility of this pattern, but prefer to lighten the burden on your consumers, check out our next pattern.

## Decorator

See the [demo](https://webcomponents.dev/edit/9XdUWIy1wjJqyLFyii0p/src/index.ts?branch=decorator%40wfVpXdsTYhf9mY2slxW4t2Y2SjC3&p=stories) on webcomponents.dev.

[Clone the code](https://github.com/westbrook/testing-a11y/blob/decorator) on GitHub.

```html
<testing-a11y>
    <label>Label</label>
    <input />
    <div>Description</div>
</testing-a11y>
```

The decorator pattern takes the wrapper pattern, and, like its name would suggest, decorates the provided HTML with the required attributes in order to deliver the pattern accessibly. When decorating HTML that is slotted into your custom element from the outside, it is important to remember that the owner of that code (the application or component above) may have expectations as to the state of that DOM with which it is best not to interfere. In that way, our `<testing-a11y>` element, in this case, will only apply the IDs needed to complete our accessibility contract when IDs are not already available on the element(s) in question. With the possibility, too, that any required aria attributes might already have associations applied to them, the element "conditions" those attributes into the ID reference list of those attributes rather than setting them to the decorated IDs only. This is a useful pattern in a number of contexts, and can be achieved with the following helper methods:

```js
export function conditionAttributeWithoutId(
    el: HTMLElement,
    attribute: string,
    ids: string[]
): void {
    const ariaDescribedby = el.getAttribute(attribute);
    let descriptors = ariaDescribedby ? ariaDescribedby.split(/\s+/) : [];
    descriptors = descriptors.filter(
        (descriptor) => !ids.find((id) => descriptor === id)
    );
    if (descriptors.length) {
        el.setAttribute(attribute, descriptors.join(' '));
    } else {
        el.removeAttribute(attribute);
    }
}

export function conditionAttributeWithId(
    el: HTMLElement,
    attribute: string,
    id: string | string[]
): () => void {
    const ids = Array.isArray(id) ? id : [id];
    const ariaDescribedby = el.getAttribute(attribute);
    const descriptors = ariaDescribedby ? ariaDescribedby.split(/\s+/) : [];
    const hadIds = ids.every((currentId) => descriptors.indexOf(currentId) > -1);
    if (hadIds) return function noop() {};
    descriptors.push(...ids);
    el.setAttribute(attribute, descriptors.join(' '));
    return () => conditionAttributeWithoutId(el, attribute, ids);
}
```

An element can `conditionAttributeWithId` and then cache the returned `conditionAttributeWithoutId` method to clean up at a later time, all without worrying about overwriting or removing values important to the parent context.

Beyond that, this is a rather naive example of decorating DOM in this way and assumes the first `<input>` that is slotted into it is _the_ input should be decorating, and the same with the first `<label>` element. Any other non-`<input>` and non-`<label>` element that it is provided is there to describe the input. However, it does nothing to ensure those are the only `<input>` or `<label>` elements that it receives or that it displays. Those elements would deliver content into the accessibility tree that is currently unmanaged, and any production-ready implementation of this pattern would benefit from additional validation to ensure that didn't happen. If this level of flexibility and the validation required to manage it seem uncomfortable, take a look a how our next pattern locks down the content our custom element can deliver.

## Emitter

See the [demo](https://webcomponents.dev/edit/9XdUWIy1wjJqyLFyii0p/src/index.ts?branch=emitter%40wfVpXdsTYhf9mY2slxW4t2Y2SjC3&p=stories) on webcomponents.dev.

[Clone the code](https://github.com/Westbrook/testing-a11y/tree/emitter) on GitHub.

```html
<testing-a11y
    label="Label"
    description="Description"
></testing-a11y>
```

Turn the decorator pattern up to 11 and you end up with the emitter pattern. As you see in the HTML sample above, the consuming developer no longer has to structure any of their own HTML to be slotted into the `<testing-a11y>` element. The emitter pattern relied on attributes to supply the accessible content that it will deliver and then renders the accessible HTML from that data. This approach very closely resembles patterns you may have seen in the JSX contexts of other approaches to componentizing UI. The main difference is that the accessible HTML will be rendered _inside_ of the `<testing-a11y>` element as opposed to in the position marked by the call to a `<TestingA11y>` function in JSX.

### The Decorator Pattern Plus

At the intersection of the emitter pattern and the decorator pattern is the decorator pattern plus, which I have [written about before](https://medium.com/@westbrook/decorator-pattern-plus-816eefc89824). It's crazy to think that it's more than three years old now, but it still does a great job of introducing what would otherwise be a sixth pattern to cover for this article. Pair the concepts therein with the concepts above, both in regards to testing and relating `<input>` elements to label and description content and you might find the accessibility pattern for your next custom input element!

### Panelist projects leveraging this technique
<a name="panelist-projects-leveraging-this-technique-1" id="panelist-projects-leveraging-this-technique-1"></a>

**Lion**
The Lion library leverages a form of Decorator Pattern Plus in that it can either emit DOM based on the attributes or properties that it is provided or accept content for the various responsibilities slotted into its `<lion-input>` element from the outside.

```html
<lion-input
    label="Label"
    help-text="Description"
></lion-input>

<!-- OR -->

<lion-input>
    <div slot="label">Label</div>
    <input slot="input" />
    <div slot="help-text">Description</div>
</lion-input>
```

This is powered by their [`FormControlMixin`](https://github.com/ing-bank/lion/blob/master/packages/form-core/src/FormControlMixin.js) that makes the decoration or emission of light DOM content nice and uniform across their library.

**Vaadin Web Components**
Having mentioned as part of the panel that Lion had a lot of influence on their library, I'm unsurprised to see the Vaadin team also leveraging a form of Decorator Pattern Plus as well. Here, too, you can create a `<vaadin-text-field>` from attributes/properties or slotted content.

```html
<vaadin-text-field
    label="Label"
    helper-text="Description"
></vaadin-text-field>

<!-- OR -->

<vaadin-text-field>
    <div slot="label">Label</div>
    <input slot="input" />
    <div slot="helper">Description</div>
</vaadin-text-field>
```

Here Vaadin leverages the [reactive controller pattern](https://lit.dev/docs/composition/controllers/) popularized by the [Lit team](https://lit.dev/) to manager various parts of this pattern. [Label content](https://github.com/vaadin/web-components/blob/master/packages/field-base/src/labelled-input-controller.js), [description content](https://github.com/vaadin/web-components/blob/master/packages/field-base/src/field-aria-controller.js), as well as the [`<input>` element](https://github.com/vaadin/web-components/blob/master/packages/field-base/src/input-controller.js) itself are each managed in a way that is easily sharable across the library.

In both of these cases, you are also given the option to choose what things you want to supply and where, while still having them bound to the accessibility tree correctly. This can lend a nice level of freedom to developer consuming your custom form elements.

## Outside-in

See the [demo](https://webcomponents.dev/edit/9XdUWIy1wjJqyLFyii0p/src/index.ts?branch=outside-in%40wfVpXdsTYhf9mY2slxW4t2Y2SjC3&p=stories) on webcomponents.dev.

[Clone the code](https://github.com/Westbrook/testing-a11y/tree/outside-in) on GitHub.

```html
<testing-a11y>
  <div slot="label">Label</div>
  <div slot="description">Description</div>
</testing-a11y>
```

In this approach, there is content important to the accessibility story of the element on both the outside and the inside of our `<testing-a11y>` element. Inside, by default, consumers of this element are provided an `<input>` element, and from the outside content is addressed to `label` and `description` slots for their content to be associated with said `<input>` appropriately. Much like we saw with the emitter approach above, this allows a consuming developer to focus directly on providing the content they would like to deliver while the `<testing-a11y>` element manages all of the accessible relations. This pattern goes one step further in not altering DOM contexts that it does not own, which can ensure eager rendering technologies employed at the parent application or element level will not interfere with the UI our custom element delivers.

### ID references DO NOT pass through shadow boundaries

This is the first technique we've looked at together where there is content important to delivering the accessibility of the pattern separated by shadow boundaries. In association with that, you'll notice that we are no longer supplying IDs directly on the element containing the label and description text content. This is because the ID reference created by the `for` attribute on a `<label>` element and the `aria-describedby` attribute on an `<input>` DO NOT pass through shadow boundaries. To avoid this reality, we've wrapped the `<slot>` elements onto which we are projecting this content from the light DOM into our shadow DOM in elements that hold these references. Content projected into a custom element in this way will be attributed to those wrapping elements when the browser constructs the accessibility tree from this DOM to pass to the screen reader clearly delivering the content of this UI to the users they support.

### Panelist projects leveraging this technique
<a name="panelist-projects-leveraging-this-technique-2" id="panelist-projects-leveraging-this-technique-2"></a>

**Carbon Web Components**
We can see a full investment into the outside-in pattern in Carbon Web Components' [`<bx-input>` element](https://github.com/carbon-design-system/carbon-web-components/blob/main/src/components/input/input.ts), including some additional slots for content beyond that covered herein.

```html
<bx-input>
  <div slot="label-text">Label</div>
  <div slot="helper-text">Description</div>
</bx-input>
```

This allows the `<bx-input>` to fully leverage the accessibility relationship created by the outside-in pattern for the [`label-text` content](https://github.com/carbon-design-system/carbon-web-components/blob/aaaa893f15d1ab1dc08843fdf1261f02f249d6dd/src/components/input/input.ts#L212-L214). However, when taking a closer look, you'll see that the [`helper-text` and `validity-message` content](https://github.com/carbon-design-system/carbon-web-components/blob/aaaa893f15d1ab1dc08843fdf1261f02f249d6dd/src/components/input/input.ts#L233-L238) is not currently associated with the `<input>` element.

**Spectrum Web Components**
In order to attach description content to form elements, including the `<sp-textfield>` element in the Spectrum Web Components library, a `help-text` slot is surfaced. 

```html
<sp-textfield>
  <div slot="help-text">Description</div>
</sp-textfield>
```

This leverages the pattern outlined above very closely and expands to with a technique called [Stacked Slots](https://dev.to/westbrook/who-doesnt-love-some-s-3de0) that allows you to easily manage multiple pieces of description content based on the validity of the `<sp-textfield>` element. Even after all this time, I find that the patterns made available around slotted content and ensuring the accessibility of content leveraging shadow roots has much exploration to be had!

**UI5 Web Components**
In conjunction with a visual design decision that delivers extra content about the `<input>` element in a "popover", the `<ui5-input>` element from UI5 Web Components leverages a `valueStateMessage` slot similar to this pattern. *Notice that the `value-state` attribute must be set for content supplied in this manner to be displayed. This attribute accepts `Error`, `Information`, and `Warning` in order to display this content at various visual severity levels.*

```html
<ui5-input value-state="Information">
  <div slot="valueStateMessage">Description</div>
</ui5-input>
```

However, to achieve the delivery of the content via a popover, there is some additional machinery that goes into this implementation. By default, the text content applied to the `valueStateMessage` text is [duplicated into the shadow root](https://github.com/SAP/ui5-webcomponents/blob/8822f66438d1bea075866a70445b42487a347f09/packages/main/src/Input.js#L1302-L1312) of the `<ui5-input>` element and associated to the `<input>` via a [computed `aria-describedby` attribute](https://github.com/SAP/ui5-webcomponents/blob/8822f66438d1bea075866a70445b42487a347f09/packages/main/src/Input.js#L1276) for screen readers. When the `<ui5-input>` element is focused, any content supplied to the `valueStateMessage` slot will then be copied just in time into a popover for visual delivery. 

## Snowflakes

See the [demo](https://webcomponents.dev/edit/9XdUWIy1wjJqyLFyii0p/src/index.ts?branch=snowflakes%40wfVpXdsTYhf9mY2slxW4t2Y2SjC3&p=stories) on webcomponents.dev.

[Clone the code](https://github.com/Westbrook/testing-a11y/tree/snowflakes) on GitHub.

```html
<div>
    <testing-a11y-label for="input">Label</testing-a11y-label>
    <testing-a11y-input id="input"></testing-a11y-input>
    <testing-a11y-help-text for="input">Description</testing-a11y-help-text>
</div>
```

Everyone wants to be different, everyone wants to be unique, and sometimes custom elements feel this same way, too. To support them in this endeavor, here we outline what it might look like to make a fully custom implementation of each of the elements found in the raw HTML.

- `<testing-a11y-label>` replaces the native `<label>` element and take on the responsibility of both the focus forwarding that we confirmed in our test code, but also features its own `for` attribute that must be powered by custom JS.
- `<testing-a11y-input>` replaces the native `<input>` element and adds some simplicity by not requiring the `aria-describedby` attribute any longer.
- `<testing-a11y-help-text>` helps clarify the anonymous nature of the `<div>` we had previously leveraged for this content and also features its own `for` attribute. We'll investigate how the lack of an `aria-description` attribute in the platform makes managing this `for` attribute different than the one on our `<testing-a11y-label>` element below.

### Pretending to be a native element

One of the key characteristics of the snowflake pattern is that you are making custom elements that mimic the native behavior of existing HTML elements rather than leveraging them directly, decorating them, or extending (not likely ever possibly without polyfilling in Safari) then. This means you'll need to be conscious of the capabilities you were otherwise getting "for free" in those native elements. One should be apparent by the use of the `for` attribute in our custom label and help text elements above. Both `<testing-a11y-label>` and `<testing-a11y-help-text>` will need to duplicate the ID reference established in native `<label>` elements by this attribute. In this pattern, the `for` attribute points to an element that could be an actual form field, and you'll see code to support that possibility, but knowing that our `<testing-a11y-input>` encapsulated its form element within its shadow DOM, we'll also need to prepare a path to keep the content relationship between two elements separated by a shadow boundary live.

#### Responding to the "for" attribute

Part of the power that custom elements surface for developers in the lifecycle methods with which they can hook into browser native changes in our elements. Two of these are `observedAttributes` and `attributeChangedCallback` which allow us to [observe attribute changed](https://developers.google.com/web/fundamentals/web-components/customelements#attrchanges). With them we can easily react to changes in the `for` attribute on our custom label and help text elements to ensure that those elements are appropriately associated to the element referenced thereby. Take a closer look at how we do that in `<testing-a11y-label>`:

```js
async resolveForElement() {
    // House keeping for when the value of `for` changes from one ID to another.
    if (this.conditionLabel) this.conditionLabel();
    if (this.conditionLabelledby) this.conditionLabelledby();
    if (!this.for) {
        delete this.forElement;
        return;
    }
    // [1] Resolution of the element referenced by the ID provided as `for`. This resolution happens in the DOM tree in which the `<testing-a11y-label>` element exists, so the referenced element will need to exist there as well.
    const parent = this.getRootNode() as HTMLElement;
    const target = parent.querySelector(`#${this.for}`) as LitElement & { focusElement: HTMLElement };
    if (!target) {
        return;
    }
    if (target.localName.search('-') > 0) {
        await customElements.whenDefined(target.localName);
    }
    if (typeof target.updateComplete !== 'undefined') {
        await target.updateComplete;
    }
    // [2] Noralization of the referenced element as the referenced host or an element available via the `focusElement` property on that host (for cross shadow boundary referencing).
    this.forElement = target.focusElement || target;
    if (this.forElement) {
        const targetParent = this.forElement.getRootNode() as HTMLElement;
        if (targetParent === parent) {
            // [3a] Application of `aria-labelledby` for elements in the same DOM tree.
            this.conditionLabelledby = conditionAttributeWithId(this.forElement, 'aria-labelledby', this.id);
        } else {
            // [3b] Application of `aria-label` for elements separated by shadow boundaries.
            this.forElement.setAttribute('aria-label', this.labelText);
            this.conditionLabel = () => this.forElement?.removeAttribute('aria-label');
        }
    }
}
```

Looking specifically at the numbered comments above:

1. For performance reasons this code requires that referenced elements live in the same DOM tree. If based on the requirements of your application this might be something you could relax. As seen in 5b, there is already support for associating content across shadow boundaries, so whether you step up through the various tree to the `document` or choose to resolve the `forElement` via alternate means (possibly accepting an actual element reference in the JS scope) you should be fully prepared to label that element with this code.
2. The choice to resolve `target.focusElement || target` for the `forElement` likely restricts this approach to native form elements and custom form elements that have bought into this technique, which could be seen as unfortunate. However, it does closely mimic the [Cross-root Aria Delegation](https://leobalter.github.io/cross-root-aria-delegation/) specification that is currently under development and widely agreed to across the various browser vendors.
3. Here we gate between supporting elements in the same DOM tree and those separated by shadow boundaries. This ensures that our accessibility story continues to be delivered on even though native ID references are not able to bridge this divide themselves.

When attempting to apply this same pattern to `<testing-a11y-help-text>` you'll quickly discover that there is not `aria-description` attribute. Because of this, associating our help text across shadow boundaries is a little more complex:

```js
const proxy = document.createElement('span');
proxy.id = 'complex-non-reusable-id';
proxy.hidden = true;
proxy.textContent = this.labelText;
this.forElement.insertAdjacentElement('afterend', proxy);
const conditionDescribedby = conditionAttributeWithId(this.forElement, 'aria-describedby', 'complex-non-reusable-id');
this.conditionDescribedby = () => {
    proxy.remove();
    conditionDescribedby();
}
```

Here we create a proxy element to inject into the DOM tree on the other side of the shadow boundary with which to associate the help text provided to our form element. This does mean that our `<testing-a11y-help-text>` will be injecting DOM into a rendering scope that is does not own and it is important to keep in mind the limitations and dangers of doing so when choosing your path forward in this area. However, even when separated by this shadow boundary if you (or the same developer/library) own the elements on both sides of the divide these realities can be easily handled and the accessibility tree shaped from your content should be both stable and reliable.

#### Observing text content

When reaching across shadow boundaries to manage these label or description relationships, one added responsibility is insuring that the text content applied to the form element across the shadow boundary is kept up-to-date. Our elements will meed to observer their mutations in order to do this.

```js
public connectedCallback(): void {
    super.connectedCallback();
    if (!this.observer) {
        this.observer = new MutationObserver(() => this.resolveForElement());
    }
    this.observer.observe(this, { characterData: true, subtree: true, childList: true });
}

public disconnectedCallback(): void {
    this.observer.disconnect();
    super.disconnectedCallback();
}

private observer!: MutationObserver;
```

The config of `{ characterData: true, subtree: true, childList: true }` ensures that the observer will trigger on all changes to the value of `el.textContent`. When that content changes it needs to be pushed over the shadow boundary into the other DOM tree so that the accessibility tree can be built with the expected relationships.

### Panelist projects leveraging this technique
<a name="panelist-projects-leveraging-this-technique-3" id="panelist-projects-leveraging-this-technique-3"></a>

**Spectrum Web Components**
This pattern is leveraged specifically for the `<sp-field-label>` element in Spectrum Web Components to deliver label content to `<sp-textfield>` elements when finishing the `<input>` interface we've explored herein.

```html
<div>
    <sp-field-label for="input">Label</sp-field-label>
    <sp-textfield id="input"></sp-textfield>
</div>
```

In the Spectrum Web Components library the `<sp-field-label>` element uses [almost line for line](https://github.com/adobe/spectrum-web-components/blob/85e525827a8aabcb4ebf441f05b7e1789b590b8b/packages/field-label/src/FieldLabel.ts#L82-L107) the approaches outlined above so that it can be leveraged in partnership with other native form elements or custom form element surfacing a `focusElement` property to provide intentional access to specific children in the elements shadow DOM.

**UI5 Web Components**
Similarly, the `<ui5-label>` element in UI5 Web Components also leverages this technique to a degree.

```html
<div>
    <ui5-label id="label" for="input">Label</ui5-label>
    <ui5-input id="input" accessible-name-ref="label"></ui5-input>
</div>
```

Here they've baked `for` attribute management into the [`<ui5-label>` element](https://github.com/SAP/ui5-webcomponents/blob/16246a8ec08f4f4f287e4dcac53266da6d4bc60d/packages/main/src/Label.js#L142-L147) and resolution from the form element to the label element via the `accessible-name-ref` attribute as part of their [AriaLabelHelper utility](https://github.com/SAP/ui5-webcomponents/blob/master/packages/base/src/util/AriaLabelHelper.js). Yet another example as to how we're really only scratching the surface as to how you could ship accessible UI with shadow roots by looking at the handful of techniques included in this article.

# In memoriam

If you hitherto thought that it was not possible to make shadow DOM accessible, thanks for sticking around this far to have that misunderstanding cleared up.

If you understood how shadow DOM could be accessible, thanks for sticking around this far to hear about ways that I understand to do so.

Hopefully everyone has a couple of extra tools for making their next web component, or one they're already shipping, even more accessible. But, remember, this is just a couple of the possibilities! If you've got other techniques that you know, use, or love, please, please share them in the comments so that whether it's me, or the next developer, the community at large can put them in their tool belt as well. 

Or even better, if you see something I missed, something I did wrong, or something you're want to know more about, get that conversation going. The only way we can all get better at delivering accessible UIs is by making it central to the conversation of delivering UIs. Sharing what we know, asking questions about what we don't, and pushing the envelope in every direction is a big part of that. I look forward to seeing it below!

# In the next life...

Like I mentioned at the start, there are MANY concepts around shipping a complete web component that we have not covered in this conversation. Including:
- content styling
- form association
- state management
- validation

Each and every one of these topics and more would make a great follow up article expanding on these above patterns in any one of those directions. I can't make any promises, but I'll do my best...if you're interested in it, I'd love to partner, support, cheer you on as it comes together!

On top of that, there are some more varied and complex patterns for accessibility with web component that could be useful to dig into, as well. In particular, the [Vaadin panel](https://www.youtube.com/watch?v=xz8yRVJMP2k&feature=youtu.be) touched on the super useful combobox patterns currently begin shipped by various products from other panelists which is currently raising up the backlog of [Spectrum Web Components](https://opensource.adobe.com/spectrum-web-components/). Sharing thoughts on how to bring that experience to the web with custom elements and shadow DOM might be just the nudge needed to actually get development of that pattern finished.

Let's keep talking about accessibility. Let's keep making our UIs and components accessible. Let's find new and better patterns for doing it, together. See you next time.

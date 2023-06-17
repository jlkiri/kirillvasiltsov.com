---
title: Unlocking the power of Svelte Actions
date: "2020-10-19"
spoiler: Make development easier with Svelte actions. Create custom events, extend DOM attributes, share animation logic and more.
language: en
tags:
  - svelte
---

This is a text version of my [Svelte Summit 2020 talk](https://sveltesummit.com/). In this article I will show you how to use Svelte Actions and why they make development with Svelte better.

## What are Svelte Actions?

Actions are essentially functions that are executed when an element is **mounted**. In other words, as soon as your HTML element like a `button` appears in the DOM, this function is called with the element as its argument. What's inside the function is entirely up to you. That's literally all there is to a Svelte Action and it might not sound very useful now, but there is actually quite a lot we can do with them. Maybe you are familiar with React Hooks - Hooks and Actions are similar in that both allow to share non-visual logic. If you have ever seen amazing things done with hooks, you can expect the same from Actions.

Let's jump straight to some examples.

## Chat input focus example

You may have seen [the example of a chat with Eliza bot](https://svelte.dev/tutorial/update) in the Svelte tutorial. I've modified it a little bit. Here's the [REPL](https://svelte.dev/repl/8b7d235f5d8d4042943c576107db12a9?version=3.29.0) that you can play with. Inside, you can see a file named `autofocus.js`. It is a plain JS file and inside there is a Svelte Action:

```js
export function autofocus(node) {
  node.focus();
}
```

That's it. This is a legitimate Svelte Action! In the main file (`App.svelte`) I import this function and _use_ it on the `input` element like this:

```svelte
<input use:autofocus on:keydown={handleKeydown}>
```

Now the `input` element is auto-focused when we load the page. Moreover, you can `use` this action on any other `input` you have in your app that you want to autofocus on mount.

Pretty awesome. You might think: "can't we do the same with something like an `Autofocusable` component"? And the answer is no. You could in theory do something similar, but in any case the component is limited to only one kind of DOM element. Either it is an autofocusable `input` or `button`. You could make it a simple wrapper with a `<slot>` but we don't need that one HTML element which only functions as a wrapper.

Moreover this approach has another big disadvantange:

- What if we wanted to reuse some other, independent functionality?

Maybe I want my `input` to also use some common input validation logic with `use:validate`? Where should I put it? Would it fit into the `Autofocusable` component? I don't think so.

## Extend DOM attributes

One interesting use case I've found is extending the semantics of DOM attributes. For example `input`s that have type `range` can have a step: if the step is 2 then the value increases by 2, if it is 4 then by 4 etc. But the problem is that the step is by default fixed. What I would like to do is to use one step value if the current `input` value is below 6, another step if it is below 8 etc. Fortunately this is very easy to do and share between different inputs with Actions. Here's the [REPL that shows how to do it](https://svelte.dev/repl/42c9753e490d455793376947736abd7f?version=3.29.0). It uses an `input` to dynamically update padding on the red box below.

You can see an `input` that uses the action:

```svelte
<input type="range" use:step={{ steps, default: 8 }} />
```

The Action expects from us an array that looks something like this:

```js
const steps = [
  { below: 6, step: 1 },
  { below: 12, step: 2 },
  { below: 24, step: 4 },
];
```

And here is the body of our Action:

```js
export function step(node, config) {
  function adjustStep(inputVal) {
    for (const condition of config.steps) {
      if (inputVal < condition.below) {
        return condition.step;
      }
    }

    return config.default;
  }

  function handleRangeChange(e) {
    node.step = adjustStep(e.target.value);
  }

  node.addEventListener("input", handleRangeChange);
}
```

It is very simple. All it does is loop through the array and look at the member's `below` field. If the current `input` value satisfies the condition then the corresponding step is applied and the function immediately `return`s.

To be more precise, the value updated is more like a **level** that some other function uses to calculate the real `padding` value. Does it sound familiar? I discovered it when I needed to dynamically set Tailwind CSS class names - such as `p-4` or `p-8`. As you might know there is no `p-7` or `p-9` class in Tailwind. So if we used a default `input`, at some point our CSS would break because it would try to apply the non-existent `p-9` class.

## Interim summary

Hopefully, you are starting to see the most obvious advantages of Svelte Actions:

- You can reuse logic that can be applied to HTML elements of different kinds (non-visual logic, e.g. autofocus example above)
- Customize logic that is applied to DOM elements of the same kind (e.g. extend DOM attributes like in the `step` example above)
- Library authoring

It is a good fit for libraries, because many libraries need to do some work both on mount and after every update. Using callbacks like `afterUpdate` only to call some library function feels wrong - these are implementation details that must be hidden. Ideally you should be able to just import the library and then use it with the `use` keyword.

## CSS properties

In Svelte you cannot update CSS dynamically when state changes. There are some CSS-in-JS libraries that allow you to do that but otherwise it is impossible. Normally, if you need to update CSS you do it through different class names. But you can get quite close to directly updating CSS with Actions!

Here's the [REPL that shows how to do it](https://svelte.dev/repl/ddffbe09e1b94ab990ed3fc61294d96d?version=3.29.0).

There is a `css.js` file which contains the Action. We import it and `use` it on a couple of elements like this:

```svelte
<div use:css={{ color }} class="container">
	<h1 use:css={{ fontSize }}>Hello world!</h1>
```

At the top level we have two variables (with default values) which we are going to actually update:

```js
let fontSize = 2;
let color = "#000";
```

That's not all. We also need at least something in our CSS that can be set from outside. CSS custom properties are a great fit for this! So inside the `<style>` tag we put some CSS custom properties inside the classes that are applied to the same elements that our `css` action is applied to.

```css
h1 {
  --fontSize: ;
  font-size: calc(var(--fontSize) * 1rem);
  color: inherit;
}

.container {
  --color: black;
  color: var(--color);

  display: grid;
  place-items: center;
  row-gap: 16px;
}
```

Here's what happens when we update `fontSize` and `color` variables:

1. Variables get updated
2. The new values are passed to `use:css` action
3. The special `update` function that our action returns is called with the new values
4. It uses [`setProperty`](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleDeclaration/setProperty) function to set the value of the CSS property for that element

The `update` function that you can return from an action allows you to repeat the same thing you did on mount when parameters (if defined) are updated.

This is what the `css` looks like:

```js
export function css(node, properties) {
  function setProperties() {
    for (const prop of Object.keys(properties)) {
      node.style.setProperty(`--${prop}`, properties[prop]);
    }
  }

  setProperties();

  return {
    update(newProperties) {
      properties = newProperties;
      setProperties();
    },
  };
}
```

It just loops through the argument's object keys (such as `fontSize`) and uses their name (and value) to set the corresponding CSS custom property.

Pretty simple but very powerful. It allows us to update any CSS value we want through just a bit of indirection which is CSS custom variables.

## Animations

Finally, I think Actions are great for animations, because you will definitely need to apply the same logic to many different elements. One of my favorite examples are FLIP animations, where a change in DOM position can be animated. For example shuffling a list of items. I will not dive deep into the topic in this article: I've written about some techniques in [this article about FLIP animations in React](https://css-tricks.com/everything-you-need-to-know-about-flip-animations-in-react/) and in [this article about how to create spring animations with Web Animation API](https://www.kirillvasiltsov.com/writing/how-to-create-a-spring-animation-with-web-animation-api/). Although they are not about Svelte, at the end of the day it all boils down to manipulating the HTML element directly. And Svelte Actions are a great place to do it.

If you want to take a look at what such an Action might look like, here'a [REPL I used in the Svelte Summit talk](https://svelte.dev/repl/fc3332d390ff4bbda09ce6836733acd5?version=3.29.0).

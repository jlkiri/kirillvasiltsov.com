---
title: "When more functions = less code"
date: "2020-06-20"
spoiler: How abstraction makes it easier to minimize your code
language: en
tags:
  - programming
  - post
---

Normally, I don't think (and don't have to think) much about minimization. I just let my bundler and plugins do their job and enjoy the result. But when developing my own library, I wondered whether there was something I could do to shave some kilobytes. The answer was **abstraction**.

## Unminimizable things

Look at this line of code:

```javascript
export const button = document.getElementById("button");
```

How do you think it will be minimized? Well, here's how Rollup with minimal configuration (only a `terser` plugin) does it:

```javascript
"use strict";
Object.defineProperty(exports, "__esModule", { value: !0 });
const e = document.getElementById("button");
exports.button = e;
```

You can see how only your `button` got minimized into short `e`. But both `document` nor `getElementById` remained in their original form. This is not something surprising, because:

a) `document` **must** be referred as `document`

b) if you minimize property and method names, accessing them will fail because no such property or method exists on the object

This is a huge disappointment, because if your page uses hundreds of DOM API calls like these, then you will see all of them as is in the output. Is there anything we can do to make it more minimizable?

Yes, and that is abstraction.

## Abstract to minimize

It is very easy to abstract `document.getElementById` into a function with a descriptive name:

```javascript
const getById = (id) => document.getElementById(id);
```

Now, imagine we have a file will lots of `getById` calls. What will the minimized output look like?

Original:

```javascript
export const a = getById("a");
export const b = getById("b");
export const c = getById("c");
export const d = getById("d");
export const e = getById("e");
export const f = getById("f");
export const g = getById("g");
```

Minimized:

```javascript
"use strict";
Object.defineProperty(exports, "__esModule", { value: !0 });
const e = (e) => document.getElementById(e),
  r = e("a"),
  p = e("b"),
  x = e("c"),
  c = e("d"),
  n = e("e"),
  d = e("f"),
  i = e("g");
// ...
```

As you can see, the `getById` function got minimized into a very short `e` and then used as `e` througout the code. So, while `document.getElementById` is 23 bytes, `e` is only 1 byte. That's **23 times less**! Of course this doesn't mean that your real code is going to be 23 times less if you use this trick, because there's a lot of other stuff that is properly minimized. But in my experience, if your code heavily uses DOM methods like these, you can **expect almost the twofold difference** between the version with abstraction and without it.

This does not apply only to DOM methods. Another example is `Object` methods like `keys` or `values` or `entries`. All of them are not going to be minimized if you use them directly as e.g. `Object.entries`. If you use a lot of these methods in your code, it might be better to abstract it to a function:

```javascript
const getEntries = (obj) => Object.entries(obj);
const getRect = (el) => el.getBoundingClientRect();
// etc.
```

This way they will likely to be also minimized to something like `g` or `e` or whatever.

## Note on gzip

Interestingly, when you compress two outputs with gzip, the difference between them becomes much smaller. Still, on large files the difference can easily reach 50KB and more - enough to have a noticeable impact on your page load. If you are interested, you can [learn more about gzip here](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/optimize-encoding-and-transfer), [here](https://www.youtube.com/watch?v=whGwm0Lky2s&feature=youtu.be&t=14m11s) and [here](https://css-tricks.com/the-difference-between-minification-and-gzipping/).

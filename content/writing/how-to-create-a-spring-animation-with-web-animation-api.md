---
title: "How to create a spring animation with Web Animation API"
date: "2020-07-27"
spoiler: A step by step tutorial to smooth spring animations in Javascript
language: en
tags:
  - programming
  - post
  - animation
---

In this article I explain how to create animations with [Web Animation API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Animations_API) using **springs** (or rather, physics behind them).

<p align="center"><img src="https://dev-to-uploads.s3.amazonaws.com/i/6eyzyu1v0gyp04rl8e94.gif" /></p>

Spring physics sounds intimidating, and that's what kept me from using it in my own animation projects. But as [this awesome article by Maxime Heckel](https://blog.maximeheckel.com/posts/the-physics-behind-spring-animations) shows, you probably already know some of it and the rest isn't very complicated. If you haven't read the article yet, you should read it now, because everything below assumes you know the principles. If you are not familiar with Web Animation API, [start here](http://danielcwilson.com/blog/2015/07/animations-part-1/).

## Quick recap

For convenience, here's a quick recap:

- Springs have _stiffness_, _mass_ and a _damping ratio_ (also length but it is irrelevant here).
- One force that acts on a spring when you displace it is:

```plain
F = -k * x // where k is stiffness and x is displacement
```

- Another force is _damping force_. It slows the spring down so it eventually stops:

```plain
F = -d * v // where d is damping ratio and v is velocity
```

- If we know acceleration and a time interval, we can calculate speed from previous speed:

```plain
v2 = v1 + a*t
```

- If we know speed and a time interval, we can calculate position from previous position and speed:

```plain
p2 =  p1 + v*t
```

## Implementation

Here's the Codesandbox that shows the final result. You can play with it and change some default parameters.

<iframe
     src="https://codesandbox.io/embed/simple-spring-animation-x2svy?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="simple-spring-animation"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

### Listeners

First of all, we need some listeners.

- `mousedown` and `mousemove` to start tracking the displacement of the square
- `mouseup` to calculate and play an animation (more on that below)

This is pretty straightforward, so I am going to omit the details.

### Drag transform

Strictly speaking, we are not dragging the element using native browser API. But we want to make it look like we move it! To do that, we set a CSS transform string directly to the element on each `mousemove` event.

```javascript
function transformDrag(dx, dy) {
  square.style.transform = `translate(${dx}px, ${dy}px)`
}

function handleMouseMove(e) {
  const dx = e.clientX - mouseX
  const dy = e.clientY - mouseY
  dragDx = dragDx + dx
  dragDy = dragDy + dy
  transformDrag(dragDx, dragDy)
}
```

### Generating keyframes

Now, the most important part of the animation. When we release (`mouseup`) the square, we need to animate how it goes back to its original position. But to make it look natural, we use a _spring_.

Any animation that uses WAAPI requires a set of keyframes which are just like the keyframes you need for a CSS animation. Only in this case, each keyframe is a Javascript object. Our task here is to generate an array of such objects and launch the animation.

We need a total of 5 parameters to be able to generate keyframes:

- Displacement on x-axis
- Displacement on y-axis
- Stiffness
- Mass
- Damping ratio

In the codesandbox above we use these defaults for physical parameters 3-5: `600`, `7` and `1`. For simplicity, we assume that the spring has length `1`.

```javascript
function createSpringAnimation(
        dx,
        dy,
        stiffness = 600,
        damping = 7,
        mass = 1
      ) {
        const spring_length = 1;
        const k = -stiffness;
        const d = -damping;
        // ...
```

`dx` and `dy` are dynamic: we will pass them to the function on `mouseup` event.

A time interval in the context of the browser is _one frame_, or ~0.016s.

```javascript
const frame_rate = 1 / 60
```

To generate one keyframe we simply apply the formulas from the article above:

```javascript
let x = dx
let y = dy

let velocity_x = 0
let velocity_y = 0

let Fspring_x = k * (x - spring_length)
let Fspring_y = k * (y - spring_length)
let Fdamping_x = d * velocity_x
let Fdamping_y = d * velocity_y

let accel_x = (Fspring_x + Fdamping_x) / mass
let accel_y = (Fspring_y + Fdamping_y) / mass

velocity_x += accel_x * frame_rate
velocity_y += accel_y * frame_rate

x += velocity_x * frame_rate
y += velocity_y * frame_rate

const keyframe = { transform: `translate(${x}px, ${y}px)` }
```

Ideally we need a keyframe for _each_ time interval to have a smooth 60fps animation. Intuitively, we need to loop until the end of animation duration (duration divided by one frame length times). There's a problem, however - we don't know **when** exactly the spring will stop beforehand! This is the biggest difficulty when trying to animate springs with browser APIs that want the exact duration time from you. Fortunately, there is a workaround: loop a potentially large number of times, but break when we have enough keyframes. Let's say we want it to **stop when the largest movement does not exceed 3 pixels (in both directions) for the last 60 frames** - simply because it becomes not easy to notice movement. We lose precision but reach the goal.

So, this is what this heuristic looks like in code:

```javascript
const DISPL_THRESHOLD = 3

let frames = 0
let frames_below_threshold = 0
let largest_displ

let positions = []

for (let step = 0; step <= 1000; step += 1) {
  // Generate a keyframe
  // ...
  // Put the keyframe in the array
  positions.push(keyframe)

  largest_displ =
    largest_displ < 0
      ? Math.max(largest_displ || -Infinity, x)
      : Math.min(largest_displ || Infinity, x)

  if (Math.abs(largest_displ) < DISPL_THRESHOLD) {
    frames_below_threshold += 1
  } else {
    frames_below_threshold = 0 // Reset the frame counter
  }

  if (frames_below_threshold >= 60) {
    frames = step
    break
  }
}
```

After we break, we save the number of times we looped as the number of frames. We use this number to calculate the actual duration. This is the `mouseup` handler:

```javascript
let animation

function handleMouseUp(e) {
  const { positions, frames } = createSpringAnimation(dragDx, dragDy)

  square.style.transform = "" // Cancel all transforms right before animation

  const keyframes = new KeyframeEffect(square, positions, {
    duration: (frames / 60) * 1000,
    fill: "both",
    easing: "linear",
    iterations: 1
  })

  animation = new Animation(keyframes)

  animation.play()
}
```

Note that the `easing` option of the animation is set to `linear` because we already solve it manually inside the `createSpringAnimation` function.

This is all you need to generate a nice smooth 60fps spring animation!

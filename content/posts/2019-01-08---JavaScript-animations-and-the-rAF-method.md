---
title: JavaScript Animations & the rAF API
date: '2019-01-09T08:48:15.169Z'
template: 'post'
draft: false
slug: '/posts/animations-in-javascript/'
category: 'JavaScript'
tags:
  - 'Web Development'
description: 'Exploring the requestAnimationFrame method to achieve smooth animations.'
---

Recently, I've taken a ticket at work to declutter the frontend update
any unnecessary third party dependencies. _One_ particular API that caught my eye was
this `click` handler that consumed this particular JQuery API:

```text
function scrollToLeft(container, targetEl, duration = 750){
  /* omitted some logic for brevity */
  $(container).animate({ scrollLeft: targetElCenter }, duration);
}
```

#### So what exactly did this achieve?

The idea behind this method was to horizontally scroll the `container` DOM to center
the focused target `targetEl` element (the calculated `targetElCenter`).

Visually, think of something along these lines:
![JQuery Demo](/media/gif-scroll_left-jquery.gif)

## Defining the acceptance criteria

The main goal of this exercise is to remove unnecessary dependencies. So no additional packages here.

After inspecting the JQuery source code to figure out how `.animate()` does its magic,
I've determined the following:

**The method assigns a set of key values: the start position/state, end, and unit values.**

Each step is determined by the following factors: displacement between the start and end state, the rate of change (easing function), and the total duration defined.

In the case of our example, the container _starts_ from some horizontal scroll position, will _end_ up at another scroll position to center the selected element. The rate in which the container moves from start to finish is described with an easing function. A single iteration of this animation can last up to 400ms.

**A callback interval is invoked to transform the target element's state in step-wise methods:**

a callback method of describing the transtions is set to 13ms in order to achieve the refresh rate of approximately 60fps.

So say for a 750ms animation timeframe, there will be 45 instances ( 750 / 1000 sec \* 60 segments/sec ) in which the callback will be called to return the displacement of the key values described in (A).

Following this approach, I decided to break up the calculations and to three separate methods:

1. Develop a function for calculating the "deltas", or the displacement between the start and end point
2. Develop an easing function to describe the displacement rates, i.e. moving from point A to B
3. Develop an animation "setInterval" loop to connect methods (1) and (2).

#### Calculating the displacement from point A to B

Using `Element.getBoundingClientRect()` did the job of getting the container and the target element's width and position on the viewport.

```text
const getTargetDelta = (container, targetElement) => {
  const { x: containerLeft, width: containerWidth } = container.getBoundingClientRect();
  const { x: targetLeft, width: targetWidth } = targetElement.getBoundingClientRect();

  const containerCenter = containerLeft + containerWidth * 0.5;
  const targetOffsetCenter = targetLeft + targetWidth * 0.5;

  return targetOffsetCenter - containerCenter;
}
```

#### Finding an Easing function

Since I wished to come close to JQuery's "swing" easing function,
I decided to use the easeInOutCubic method:

```text
const easeInOutCubic = delta => {
  return delta < 0.5 ?
    2 * delta ** 2 :
    -1 + (4 - 2 * delta) * delta;
};
```

For an excellent discussion on easing functions, check out [this gist post](https://gist.github.com/gre/1650294)

#### requestAnimationFrame: setting things into motion (🥁)

Per MDN [link](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame)

<blockquote>
The `window.requestAnimationFrame()` method tells the browser that you wish to perform an animation and requests that the browser call a specified function to update an animation before the next repaint. The method takes a callback as an argument to be invoked before the repaint.
</blockquote>

`requestAnimationFrame` basically provides a more appropriate application to using JavaScript's `setInterval(callbackFn, timeMs)`:

So instead of having to set the number of 13-16ms to ideally achieve a 60fps callback rater, rAF allows me to
abstract the context of elegantly refreshing my callback displacement method.

```text
// Assign values to represent the state of the animation frames
let currentFrame = 0;
const totalFrames = durationInMs / 1000;

const animate = () => {
  const progress = currentFrame / totalFrames;

  // Add the DOM manipulation methods here
  // This is where the calculations + step-wise function will go
  manipulateTheDOM();
  currentFrame += 1;

  // conditionally call for recursion depending on the progress
  if(progress < 1) window.requestAnimationFrame(animate);
};

window.requestAnimationFrame(animate);
```

## Final product & Other thoughts

Assembling putting everything together, I've ended up with the following:

```
const scrollToLeft = (container, target, duration = 750) => {
  // assign values for the state of the animation frames
  let currFrame = 0;
  const endFrame = duration / 1000 * 60;

  // let's assign a value for the initial state of the container
  const { scrollLeft: offsetLeftInitial } = container;

  const animate = () => {
    currFrame += 1;

    const step =
      getTargetDelta(container, target) * easeInOutCubic(currFrame / endFrame)
      + offsetLeftInitial;

    // update the scroll position of the container
    container.scrollLeft = step;

    if(currFrame < endFrame) window.requestAnimationFrame(animate);
  };
  window.requestAnimationFrame(animate);

};
```

## Codepen Demo [[LINK]](https://codepen.io/larrycustodio-the-flexboxer/pen/JxKgab/)

Enjoy!

...

**TL;DR** - I wrote a bunch of a modularized methods to replace JQuery's
animate() method. Check out the CodePen for a snippet of the solution.

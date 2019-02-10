---
title: JavaScript Animations & the rAF API
date: "2019-01-09T08:48:15.169Z"
template: "post"
draft: false
slug: "/posts/animations-in-javascript/"
category: "JavaScript"
tags:
  - "Web Development"
description: "Exploring the requestAnimationFrame method to achieve smooth animations."
---

Recently, I was tasked at work to deprecate some JQuery methods.
_One_ particular API that caught my eye was this `click` handler that
consumed this particular call (condensed for the brevity):

```text
function scrollToLeft(container, targetEl, duration = 750){
  $(container).animate({ scrollLeft: targetElCenter }, duration);
}
```

#### So what exactly did this achieve?

To add context to the scope of the web app, this triggered
a "container" element to scroll horizontally, to the center
the target (child) element (per the method described above, the calculated `targetElCenter`).

If you're a visual person like myself, think of it as something of this manner:
![JQuery Demo](/media/gif-scroll_left-jquery.gif)

## Defining the acceptance criteria

Like most things on the frontend, there are tons of available ways to approach this
problem. Provided that my stretch goal was to reduce the dependency for even _more_ npm packages,
frameworks and libraries are out of the question.

After inspecting the JQuery source code to figure out how `.animate()` does its magic,
I've determined the following:

- we start by storing a set of key values: the start position/state, end, and unit values.
- an interval callback is invoked to transform the target element's state in discretised/
  step-wise methods:

  A) Said steps are determined by the displacement between the start and end state, the rate of change (easing function), and the total duration defined.

  For our example, the container _starts_ from some horizontal scroll position, will _end_ up at another scroll position to center the selected element, and will "ease" from start to end within a defined period of time. By default, the JQuery describes the "ease" as a "swing", within a period of 400ms.

  B) The setInterval method is called every 13ms to achieve the refresh rate of approximately 60fps. Back to our example, the 750ms duration means that our animation frames will be divided into into 45 segments ( 750 / 1000 sec \* 60 segments/sec ).

Following this approach, I decided to divide up the calculations and to three separate methods:

1. A dedicated function for calculating the "delta", or the displacement between the start and end point
2. An easing function to describe the rate in which the element with transtion from point A to B
3. An animation "interval" loop to put the two methods together

#### Calculating the displacement from point A to B

For this specific use case, using `Element.getBoundingClientRect()` did the job of
getting the container and the target element's width and position on the viewport.
Essentially, this allowed me to determine how much my container element will
be "travelling" to get to it's target scroll position.

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

For a good in-depth discussion on easing functions, follow [this gist post](https://gist.github.com/gre/1650294)

#### requestAnimationFrame: setting things into motion (🥁)

Per MDN [link](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame)

<blockquote>
The `window.requestAnimationFrame()` method tells the browser that you wish to perform an animation and requests that the browser call a specified function to update an animation before the next repaint. The method takes a callback as an argument to be invoked before the repaint.
</blockquote>

`requestAnimationFrame` basically provides a more versatile version of the setInterval approach:

So rather than wrapping calling our animation calls in a `setInterval()` callback & using
the magical number of 13-16ms to represent a callback rate of 60fps, rAF allows me to
abstract this context of how often to refresh the callback over to the browser.

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

As far as extensibility of this approach goes, refactoring this feature to a module would be a nice step towards added readability. Adding more conditionals would be a nice plus as well (e.g. if the container's scrollLeft position has reached 0 after a first repaint, then terminate the loop). Otherwise, the easing method as well as the animation loop should hold up well for most applications that call for JavaScript animation. The delta calculation however, was designed just for this specific use case. For other cases that call for transforming/translating, a more specialized `getTargetDelta` would be required.

Otherwise, using CSS keyframes as a means of animating elements is still much preferred. Next time, I'll do an in-depth approach to animations using ubiquitious animation JS-frameworks (Velocity, GSAP, etc) and CSS.

...

**TL;DR** - I wrote a bunch of a modularized methods to replace JQuery's
animate() method. Check out the CodePen for a snippet of the solution.

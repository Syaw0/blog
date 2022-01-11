---
layout: post
title: "Let's take a look at ResizeObserver"
date: 2022-01-09 18:21:10 +0900
categories: WebAPI
---

This is a blog post for my video [https://www.youtube.com/watch?v=XEHR-AVou4U](https://www.youtube.com/watch?v=XEHR-AVou4U)

## Syntax from MDN

Per name, ResizeObserver allows us to observe resizing of an element.

There is this [demo example on MDN](https://developer.mozilla.org/en-US/docs/Web/API/ResizeObserver), in which ResizeObserver is used to change fontSize by listening to container's size changes.

Syntax is straightforward, all we need is to set up a callback and `observe()` the target element.

```js
const resizeObserver = new ResizeObserver((entries) => {
  for (let entry of entries) {
    h1Elem.style.fontSize =
      Math.max(1.5, entry.contentRect.width / 200) + "rem";
    pElem.style.fontSize = Math.max(1, entry.contentRect.width / 600) + "rem";
  }

  console.log("Size changed");
});

resizeObserver.observe(divElem);
```

The callback is called with entries containing size info.

![](/static/resize-observer-1.png)

## Timing

ResizeObeserver callback is **triggered after layout and before paint in Event Loop model**.

For more on this, I suggest reading [this article about event loop ](https://xnim.me/blog/javascript-browser-event-loop-layout-paint-composite-call-stack) (TBH, I still cannot explain it very well though I have read it many times)

Actually about the timing, [the spec](https://www.w3.org/TR/resize-observer/#html-event-loop) has clear definition.

```
For each fully active Document in docs, run the following steps for that Document and its browsing context:

1. Recalc styles

2. Update layout

3. Set depth to 0

4. Gather active observations at depth depth for Document

5. Repeat while document has active observations

   2. Set depth to broadcast active observations.

   3. Recalc styles

   4. Update layout

   5. Gather active observations at depth depth for Document

6. If Document has skipped observations then deliver resize loop error notification

7. Update the rendering or user interface of Document and its browsing context to reflect the current state.

```

Simply speaking, **it checks all the ResizeObserver and see if callback should be run**.

Wait, what does `depth` do here?

## Infinite loop problem

Yeah, you might have already noticed it. If we change some style in the ResizeObserver callback, it might lead to another resize event and thus creating infinit loop.

And `depth` is the internal implementation to avoid this issue.

In each iteration of step 5, depth is set to new and shallower depth, so even if we try to trigger size change in the callback, it will be processed in next round if the changes are not in the children.

Reasonable! Who would do that in reality? Yes, me! Below is a modified example.

```diff
const resizeObserver = new ResizeObserver(function (entries){
+ divElem.style.width = (parseInt(divElem.style.width) + 2) + 'px';
+ h1Elem.style.color = randomColor();
  for (let entry of entries) {
    if(entry.contentBoxSize) {
      // Checking for chrome as using a non-standard array
      if (entry.contentBoxSize[0]) {
        h1Elem.style.fontSize = Math.max(1.5, entry.contentBoxSize[0].inlineSize/200) + 'rem';
        pElem.style.fontSize = Math.max(1, entry.contentBoxSize[0].inlineSize/600) + 'rem';
      } else {
        h1Elem.style.fontSize = Math.max(1.5, entry.contentBoxSize.inlineSize/200) + 'rem';
        pElem.style.fontSize = Math.max(1, entry.contentBoxSize.inlineSize/600) + 'rem';
      }

    } else {
      h1Elem.style.fontSize = Math.max(1.5, entry.contentRect.width/200) + 'rem';
      pElem.style.fontSize = Math.max(1, entry.contentRect.width/600) + 'rem';
    }
  }
  console.log('Size changed');
 });


+ const h1resizeObserver = new ResizeObserver((entries) => {
+ h1Elem.style.color = 'black';
+ });
+ h1resizeObserver.observe(h1Elem);

resizeObserver.observe(divElem);

```

We change container size and color of h1 to random color in the callback.

Since h1's font size is changed as well, its resize event triggers our newly added observer in which color is set to black.

If infinite loop is not handled, we would expect the fullscreen container to pop up instantly.

But please open the [demo link](/demos/resizeobserver/index.html) and try it yourself.

1. because of infinite loop is guarded, even though we changed the size of container, it is processed in next round, so actually we see a smooth animation of enlarging.
2. because resize event of children in the callback is processed, we see the color of h1 always be black until the last frame where h1 doesn't have size change so color is set to random.

That's it, hope this blog helps you understand ResizeObserver better, especially the internals.

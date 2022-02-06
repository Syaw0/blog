---
layout: post
title: "How does React handle empty values(null/undfined/Booleans) internally?  - React Source Code Walkthrough 18"
date: 2022-02-04 18:21:10 +0900
categories: React
image: /static/logo.png
---

We all know that [Booleans, Null, and Undefined Are Ignored](https://reactjs.org/docs/jsx-in-depth.html#booleans-null-and-undefined-are-ignored), examples like below render all same thing.

```jsx
<div />

<div></div>

<div>{false}</div>

<div>{null}</div>

<div>{undefined}</div>

<div>{true}</div>
```

But how exactly are these values being handled internally by React? Let's find it out.

These values are not components, so they only exist as children of some components, so let's have a try at [reconcileChildren()](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L277).

> This post is part of the series of me learning as well, if you find some baffled parts, I guess you can first look at my past posts or videos.

> Like above I just jump to that function because I already know it, which is hard to explain why.

In [reconcileChildren()](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L277), `mountChildFibers()` is used for first render, and `reconcileChildFibers()` is for latter renders.

But actually these 2 functions are the same with one difference of whether to track side effects or not.

([source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactChildFiber.old.js#L1366-L1367))

```js
export const reconcileChildFibers = ChildReconciler(true);
export const mountChildFibers = ChildReconciler(false);
```

From [ChildReconciler()](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactChildFiber.old.js#L267), we can see that **side effects** means "deletion".

`ChildReconciler()` consists a few closure functions under above flag, exporting the true [reconcileChildFibers()](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactChildFiber.old.js#L1260).

```js
function reconcileChildFibers(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChild: any,
  lanes: Lanes
): Fiber | null {
  if (typeof newChild === "object" && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        return placeSingleChild(
          reconcileSingleElement(
            returnFiber,
            currentFirstChild,
            newChild,
            lanes
          )
        );
      case REACT_PORTAL_TYPE:
        return placeSingleChild(
          reconcileSinglePortal(returnFiber, currentFirstChild, newChild, lanes)
        );
      case REACT_LAZY_TYPE:
        if (enableLazyElements) {
          const payload = newChild._payload;
          const init = newChild._init;
          // TODO: This function is supposed to be non-recursive.
          return reconcileChildFibers(
            returnFiber,
            currentFirstChild,
            init(payload),
            lanes
          );
        }
    }
    if (isArray(newChild)) {
      return reconcileChildrenArray(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes
      );
    }

    if (getIteratorFn(newChild)) {
      return reconcileChildrenIterator(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes
      );
    }

    throwOnInvalidObjectType(returnFiber, newChild);
  }

  if (
    (typeof newChild === "string" && newChild !== "") ||
    typeof newChild === "number"
  ) {
    return placeSingleChild(
      reconcileSingleTextNode(
        returnFiber,
        currentFirstChild,
        "" + newChild,
        lanes
      )
    );
  }

  // Remaining cases are all treated as empty.
  return deleteRemainingChildren(returnFiber, currentFirstChild);
}
```

It has 4 steps

1. handle single element type based on `$$typeof`.
2. handle array or iterators
3. non-empty string and numbers
4. the rest are treated as empty, leading to deletion of previous fiber if any.

So we can see that **null, undefined and booleans are simply ignored** when creating fiber.

What about the case when it is in an array, let's look at [reconcileChildrenArray()](https://github.com/facebook/react/blob/848e802d203e531daf2b9b0edb281a1eb6c5415d/packages/react-reconciler/src/ReactChildFiber.old.js#L750).

`reconcileChildrenArray()` has some algorithm I'd like to cover later.

Looking at the code, we see 2 places where new fibers might be created.

```js
const newFiber = updateSlot(returnFiber, oldFiber, newChildren[newIdx], lanes);

const newFiber = updateFromMap(
  existingChildren,
  returnFiber,
  newIdx,
  newChildren[newIdx],
  lanes
);
```

In `reconcileChildrenArray()`, a new linked fiber list is constructed by looping through the array items.

If newFiber is null, it is simply ignored and not put on the fiber tree.

In [updateSlog()](https://github.com/facebook/react/blob/848e802d203e531daf2b9b0edb281a1eb6c5415d/packages/react-reconciler/src/ReactChildFiber.old.js#L564) and [updateFromMap()](https://github.com/facebook/react/blob/848e802d203e531daf2b9b0edb281a1eb6c5415d/packages/react-reconciler/src/ReactChildFiber.old.js#L564), we find the similar pattern in which empty values are simply ignored and `null` is returned.

```js
function updateSlot(
  returnFiber: Fiber,
  oldFiber: Fiber | null,
  newChild: any,
  lanes: Lanes,
): Fiber | null {
  if (
    (typeof newChild === 'string' && newChild !== '') ||
    typeof newChild === 'number'
  ) {
    ...
  }

  if (typeof newChild === 'object' && newChild !== null) {
    ...
  }

  return null;
}


```

That's it. We now know how empty values are handled in React - **the are simply ignored**.

One slight problem is that actually they affects the reconciling algorithm in `reconcileChildrenArray()`, which I'll write a post about soon, stay tuned.

---
layout: post
title: "How does 'key' work? Array diffing in React - React Source Code Walkthrough 19"
date: 2022-02-04 18:21:10 +0900
categories: React
image: /static/logo.png
---

> Watch [my video on youtube](https://youtu.be/7YHnsH3KCJE) for this post.

When we render a list without adding `key` for each item, we woulld get a warning.

![](/static/array-diffing-1.png)

The reason for this is explained pretty nicely [on the homepage](https://reactjs.org/docs/reconciliation.html#recursing-on-children), but the explanation is conceptual, let's get hands dirty by looking at how extactly `key` works internally.

# reconcileChildrenArray()

We are already a bit familiar with the code base now, it should not be hard to target the reconcile function for array - `reconcileChildrenArray()` ([source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactChildFiber.old.js#L750))

> if not, you can search on my blog for past entries, or watch my [youtube video series](https://www.youtube.com/watch?v=OcB3rTln-fI&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3)

The function body is a bit intimidating, first starting with a long paragraph of comments.

```js
// This algorithm can't optimize by searching from both ends since we
// don't have backpointers on fibers. I'm trying to see how far we can get
// with that model. If it ends up not being worth the tradeoffs, we can
// add it later.

// Even with a two ended optimization, we'd want to optimize for the case
// where there are few changes and brute force the comparison instead of
// going for the Map. It'd like to explore hitting that path first in
// forward-only mode and only go for the Map once we notice that we need
// lots of look ahead. This doesn't handle reversal as well as two ended
// search but that's unusual. Besides, for the two ended optimization to
// work on Iterables, we'd need to copy the whole set.

// In this first iteration, we'll just live with hitting the bad case
// (adding everything to a Map) in for every insert/move.
```

It looks like React is comprimizing here, NOT doing `two ended optimization` because `we don't have backpointers on fibers`.

In the [previous entries]({% post_url 2022-01-08-how-does-bailout-work %}), we've already seen for children, React holds a linked list of fibers by 'sibling' rather than array.

What is `two ended optimization`? I guess it would be some algorithm which scans from end rather than start. This leads to our first problem.

# What kind of changes could happen for a list ?

Suppose we have an array.

```js
const arrPrev = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
```

After some actions, it becomes a new array. If we find some element at index `i` is different, there might have many possibilities of modifications.

1. a new element is inserted at index `i`
2. a new element is replacing the old element at index `i`
3. an existing element is moved from other index to `i`
4. an existing element is moved from other index and replaced the old element at index `i`.
5. old element `i` is removed, the next element takes the position.
6. ...

We can see it is hard to figure out which is the case without extra analysis.

Suppose we have a new array.

```js
const arrNext = [11, 12, 9, 4, 7, 16, 1, 2, 3];
```

How should we transform with minimum cost?

If we care about the minimum moves, it is similar to [Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance), but after finding the minium moves, we still need to to the transformation.

`Total Cost = Cost of analysing(reconcile) + Cost of transforming (commit changes)`

Take an extream example, what if we reversed the array ?

```js
const arrPrev = [10, 9, 8, 7, 6, 5, 4, 3, 2, 1];
```

Since each position is different, the analysing part is going to cost a lot of time to find the optimal moves, which might be better we just replace all of them.

Obviously we can take the dummiest approach - treat all different positions as replace with new items.

This helps understand why there is `two ended optimization`, think about case like below

```js
const arrPrev = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
const arrNext = [11, 12, 7, 8, 9, 10];
```

It is hard to infer from head, but if we reverse it

```js
const arrPrev = [10, 9, 8, 7, 6, 5, 4, 3, 2, 1];
const arrNext = [10, 9, 8, 7, 6, 12, 11];
```

It looks much clear now. I guess this is the `two-way optimization`.

## React's approach

In order to understand the code below better, let me explain the algorithm first.

Construct new fiber list by following step

1. try optimisitically reconcile the items with same key

   - both starts at index: 0, (headOld, headNew)
   - compare the old fiber and new element
     - if Key is the same, then reconcile the item
     - if not, break

2. for the rest
   - see if we can reuse the old fibers by `key`
     - if can, reconcile the item
     - if not, create new
   - delete the unused fibers

Let's go back to React source code. It starts with a for loop comparing old fibers and new elements.

```js
for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
  if (oldFiber.index > newIdx) {
    nextOldFiber = oldFiber;
    oldFiber = null;
  } else {
    nextOldFiber = oldFiber.sibling;
  }
  const newFiber = updateSlot(
    returnFiber,
    oldFiber,
    newChildren[newIdx],
    lanes
  );
  if (newFiber === null) {
    // TODO: This breaks on empty slots like null children. That's
    // unfortunate because it triggers the slow path all the time. We need
    // a better way to communicate whether this was a miss or null,
    // boolean, undefined, etc.
    if (oldFiber === null) {
      oldFiber = nextOldFiber;
    }
    break;
  }
  if (shouldTrackSideEffects) {
    if (oldFiber && newFiber.alternate === null) {
      // We matched the slot, but we didn't reuse the existing fiber, so we
      // need to delete the existing child.
      deleteChild(returnFiber, oldFiber);
    }
  }
  lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
  if (previousNewFiber === null) {
    // TODO: Move out of the loop. This only happens for the first run.
    resultingFirstChild = newFiber;
  } else {
    // TODO: Defer siblings if we're not at the right index for this slot.
    // I.e. if we had null values before, then we want to defer this
    // for each null value. However, we also don't want to call updateSlot
    // with the previous one.
    previousNewFiber.sibling = newFiber;
  }
  previousNewFiber = newFiber;
  oldFiber = nextOldFiber;
}
```

Oh man, what the heck is going on. Let's go step by step

```js
// this loops throught the new elements
// by keeping track of oldFiber, looks similar to two-pointer algorithm
// recall the problem where you are asked to merge two sorted array.
for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
  // because children fibers are linked by `sibling`
  // `index` is to reveal its position in the array
  // here it says:
  // ideally both pointers move ahead, meaning all elements are possibly the same
  // but if new index is lagging behind ?
  // then there is no old fiber to compare.
  // older fibers who is after the newIndex are kept.
  // TODO: take a look at this example
  // WHY this happens?
  if (oldFiber.index > newIdx) {
    nextOldFiber = oldFiber;
    oldFiber = null;
  } else {
    nextOldFiber = oldFiber.sibling;
  }

  // we now able to reconcile specific fiber to new element
  // it could be recreated, or reused from oldFiber.alternative
  // this is where `key` comes in to play, we'll cover it later
  const newFiber = updateSlot(
    returnFiber,
    oldFiber,
    newChildren[newIdx],
    lanes
  );

  // if newFiber is null, meaning we faild to do reconcilation
  // meaning what?
  if (newFiber === null) {
    // WHY this?
    if (oldFiber === null) {
      oldFiber = nextOldFiber;
    }
    break;
  }

  if (shouldTrackSideEffects) {
    // why this line means deletion ?
    if (oldFiber && newFiber.alternate === null) {
      // We matched the slot, but we didn't reuse the existing fiber, so we
      // need to delete the existing child.
      deleteChild(returnFiber, oldFiber);
    }
  }

  // We've successfully reconciled a new fiber
  // we need to mark the fiber to tell React that
  // put its DOM node to thew new index.
  lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);

  // we need to chain the fibers up right?
  // so previousNewFiber allows us to do so.
  if (previousNewFiber === null) {
    // TODO: Move out of the loop. This only happens for the first run.
    resultingFirstChild = newFiber;
  } else {
    // TODO: Defer siblings if we're not at the right index for this slot.
    // I.e. if we had null values before, then we want to defer this
    // for each null value. However, we also don't want to call updateSlot
    // with the previous one.
    previousNewFiber.sibling = newFiber;
  }
  previousNewFiber = newFiber;
  oldFiber = nextOldFiber;
}
```

A bit clearer now right ? The key point to understand this piece of code are

1. why comparing `oldFiber.index > newIdx`
2. when `newFiber` will be null.

Anyway let's continue, following part looks straightforward.

```js
if (newIdx === newChildren.length) {
  // We've reached the end of the new children. We can delete the rest.
  deleteRemainingChildren(returnFiber, oldFiber);
  if (getIsHydrating()) {
    const numberOfForks = newIdx;
    pushTreeFork(returnFiber, numberOfForks);
  }

  // yeah, per name it returns the first child
  // because reconcileChildren is run for its parent
  // it the first child is needed to set workInProgress.child = reconcileChildFibers()
  // https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L301
  return resultingFirstChild;
}
```

Continue

```js
// remember when oldFiber is null ? yes when olderFiber.index > newIndex
if (oldFiber === null) {
  // If we don't have any more existing children we can choose a fast path
  // since the rest will all be insertions.
  // WHY?
  for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
    if (newFiber === null) {
      continue;
    }
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    if (previousNewFiber === null) {
      // TODO: Move out of the loop. This only happens for the first run.
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
  }
  if (getIsHydrating()) {
    const numberOfForks = newIdx;
    pushTreeFork(returnFiber, numberOfForks);
  }
  return resultingFirstChild;
}
```

Last piece of puzzle

```js
// Add all children to a key map for quick lookups.
// OK kind of change it back to array?
const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

// Keep scanning and use the map to restore deleted items as moves.
for (; newIdx < newChildren.length; newIdx++) {
  // it will handle the move?
  const newFiber = updateFromMap(
    existingChildren,
    returnFiber,
    newIdx,
    newChildren[newIdx],
    lanes
  );
  if (newFiber !== null) {
    // what is going on here?
    if (shouldTrackSideEffects) {
      if (newFiber.alternate !== null) {
        // The new fiber is a work in progress, but if there exists a
        // current, that means that we reused the fiber. We need to delete
        // it from the child list so that we don't add it to the deletion
        // list.
        existingChildren.delete(newFiber.key === null ? newIdx : newFiber.key);
      }
    }
    // again, place child
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    if (previousNewFiber === null) {
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
  }
}

// reasonable.
if (shouldTrackSideEffects) {
  // Any existing children that weren't consumed above were deleted. We need
  // to add them to the deletion list.
  existingChildren.forEach((child) => deleteChild(returnFiber, child));
}

if (getIsHydrating()) {
  const numberOfForks = newIdx;
  pushTreeFork(returnFiber, numberOfForks);
}
return resultingFirstChild;
```

# Still feeling dizzy about it ? let's try some example with illustrations.

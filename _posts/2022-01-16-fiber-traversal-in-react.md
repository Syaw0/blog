---
layout: post
title: "How does React traverse Fiber tree internally? - React source code walkthrough 15"
date: 2022-01-16 18:21:10 +0900
categories: React
---

> You can watch my [Youtube video explanation](https://youtu.be/3nwupG2Joaw) for this post

## Fiber Tree Structure

As we've explained before, React keep a Fiber tree internally and keeps updating it. Each fiber node has 3 properties to form the structure.

1. `child`, per name, its child fiber.
2. `sibling`, per name, its sibling fiber
3. `return`, you can think of it as (kinda) parent fiber.

There is no `children`, I guess React team wants to treat the tree traversal as simple as possible, so these 3 properties would work good enough.

We might have something like this.

![](/static/fiber-traversal-1.png)

## How Fiber Tree is traversed

There is plenty of scenarios where the fiber tree needs to be traversed, for example this is how React traverse the fiber tree to mount passive effects (such as `useEffect()`) ([source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberCommitWork.old.js#L2594))

```js
export function commitPassiveMountEffects(
  root: FiberRoot,
  finishedWork: Fiber
): void {
  nextEffect = finishedWork;
  commitPassiveMountEffects_begin(finishedWork, root);
}

function commitPassiveMountEffects_begin(subtreeRoot: Fiber, root: FiberRoot) {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    const firstChild = fiber.child;
    if ((fiber.subtreeFlags & PassiveMask) !== NoFlags && firstChild !== null) {
      ensureCorrectReturnPointer(firstChild, fiber);
      nextEffect = firstChild;
    } else {
      commitPassiveMountEffects_complete(subtreeRoot, root);
    }
  }
}

function commitPassiveMountEffects_complete(
  subtreeRoot: Fiber,
  root: FiberRoot
) {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    if ((fiber.flags & Passive) !== NoFlags) {
      setCurrentDebugFiberInDEV(fiber);
      try {
        commitPassiveMountOnFiber(root, fiber);
      } catch (error) {
        reportUncaughtErrorInDEV(error);
        captureCommitPhaseError(fiber, fiber.return, error);
      }
      resetCurrentDebugFiberInDEV();
    }

    if (fiber === subtreeRoot) {
      nextEffect = null;
      return;
    }

    const sibling = fiber.sibling;
    if (sibling !== null) {
      ensureCorrectReturnPointer(sibling, fiber.return);
      nextEffect = sibling;
      return;
    }

    nextEffect = fiber.return;
  }
}
```

Because of what React does, **each node is stepped on twice**, `begin()` and `complete()`,
you can think of it as the DOM event which has capture and bubble phase, `begin()` is the `capture` phase, and `complete()` is the `bubble` phase.

Parent fiber node will `begin()` first but `complete()` latter than children.

This approach is everywhere in React source code, with postfixes of `begin` and `complete`, we must get familiar with it before we dive in.

## Simplified code

The above code could be simplified as below

```js
let nextNode = root;

function begin() {
  while (nextNode) {
    console.log("begin ", nextNode.val);
    if (nextNode.child) {
      nextNode = nextNode.child;
    } else {
      complete();
    }
  }
}

function complete() {
  while (nextNode) {
    console.log("complete ", nextNode.val);
    if (nextNode.sibling) {
      nextNode = nextNode.sibling;
      // go to sibling and begin new
      return;
    }
    nextNode = nextNode.return;
  }
}

begin();
```

You can find the full code with test data in [this gist](https://gist.github.com/JSerZANP/dd9ba9b7d5a3d2ef75f2ce8ffc78cab4).

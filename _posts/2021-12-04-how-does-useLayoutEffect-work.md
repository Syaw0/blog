---
layout: post
title: "How does useLayoutEffect() work? - React source code walkthrough 10"
date: 2021-12-04 18:21:10 +0900
categories: React
image: /static/layouteffect.jpeg
---

> You can watch my [Youtube video explanation](https://www.youtube.com/watch?v=6HLvyiYv7HI) for this post.

`useLayoutEffect()` offers a chance to run some code synchrously right after DOM is updated, let's figure out how it works.

For most of the hooks, internally there are 2 implementations: `mountXXX()` and `updateXXX()`.

- [1. how layout effects are mounted?](#1-how-layout-effects-are-mounted)
- [2. how layout effects are updated?](#2-how-layout-effects-are-updated)
- [3. when do layout effects actually get run?](#3-when-do-layout-effects-actually-get-run)
- [4. when do layout effects get cleaned up?](#4-when-do-layout-effects-get-cleaned-up)
- [5. but how cleanups get run when component is unmounted?](#5-but-how-cleanups-get-run-when-component-is-unmounted)

## 1. how layout effects are mounted?

```jsx
function mountLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null
): void {
  let fiberFlags: Flags = UpdateEffect;
  if (enableSuspenseLayoutEffectSemantics) {
    fiberFlags |= LayoutStaticEffect;
  }
  return mountEffectImpl(fiberFlags, HookLayout, create, deps);
}
function updateLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null
): void {
  return updateEffectImpl(UpdateEffect, HookLayout, create, deps);
}
```

In turn `mountEffectImpl()` and `updateEffectImpl()` are used, which are the same functions for `useEffect()`, the different is the second argument - `hookFlags`

```js
function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  currentlyRenderingFiber.flags |= fiberFlags;
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    undefined,
    nextDeps
  );
}
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,

    baseState: null,
    baseQueue: null,
    queue: null,

    next: null,
  };

  if (workInProgressHook === null) {
    // This is the first hook in the list
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // Append to the end of the list
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
```

`mountWorkInProgressHook()` creates a new empty hook and appends to the hook list of this fiber. `pushEffect()` creates an effect object on fiber's `updateQueue`, they are connected by `memoizedState`.

```js
function pushEffect(tag, create, destroy, deps) {
  const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    // Circular
    next: (null: any),
  };
  let componentUpdateQueue: null | FunctionComponentUpdateQueue =
    (currentlyRenderingFiber.updateQueue: any);
  if (componentUpdateQueue === null) {
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = (componentUpdateQueue: any);
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }
  return effect;
}
```

Notice that not all hooks are effect hooks, for example `useState()` is not. Effect means side effects, which are supposed to run, I guess that's why they are placed in `updateQueue`, since I don't see why there could not be a different place.

The `tag` property on `Effect` is important, for case of mounting layout effects, it is `HookHasEffect | HookLayout`. Also notice

That's all for mounting, it sets up `updateQueue` and creates a new `hook` for it.

## 2. how layout effects are updated?

`updateLayoutEffect()` would be something similar.

```js
function updateEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;

  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        hook.memoizedState = pushEffect(hookFlags, create, destroy, nextDeps);
        return;
      }
    }
  }

  currentlyRenderingFiber.flags |= fiberFlags;

  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    destroy,
    nextDeps
  );
}
```

Functions are run in the reconciliation phase where new fiber tree is constructed. `current` is prefix for existing stuff, `updateWorkInProgressHook()` will move the `currentWork` forward and also returns the version being created.

We can see there is change detection by `areHookInputsEqual(nextDeps, prevDeps)`, either the props changes or not, `memoizedState` is set with the effect, but the flags are different, if there is no change, then `HookHasEffect` is not added.

Thus `HookHasEffect` is important, it indicates that the effect needs to be run or not.

## 3. when do layout effects actually get run?

Good question, for an effect, either passive effect by `useEffect()` or layout effect, the function passed in is actually the create function, the closure returned by create function is the destroy (cleanup) function.

```js
const effect: Effect = {
  tag,
  create,
  destroy,
  deps,
  // Circular
  next: (null: any),
};
```

Above is the structure of `Effect`, we can see the `create` and `destroy`.

Since effects are run after DOM mutation, so they must be in commit phase, below is where it starts.

```js
function commitRootImpl(
  root: FiberRoot,
  recoverableErrors: null | Array<mixed>,
  transitions: Array<Transition> | null,
  renderPriorityLevel: EventPriority,
) {
    ....

    // The next phase is the mutation phase, where we mutate the host tree.
    commitMutationEffects(root, finishedWork, lanes);

    commitLayoutEffects(finishedWork, root, lanes);
    ...
  }
```

`commitLayoutEffects()` will try to run all the pending layout effects. It again is sort of traversing through the tree, so the [same traversal algorithm]({% post_url 2022-01-16-fiber-traversal-in-react %}) is used, let's look at the completing phase.

```js
function commitLayoutMountEffects_complete(
  subtreeRoot: Fiber,
  root: FiberRoot,
  committedLanes: Lanes
) {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    if ((fiber.flags & LayoutMask) !== NoFlags) {
      const current = fiber.alternate;
      setCurrentDebugFiberInDEV(fiber);
      try {
        commitLayoutEffectOnFiber(root, current, fiber, committedLanes);
      } catch (error) {
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
      sibling.return = fiber.return;
      nextEffect = sibling;
      return;
    }

    nextEffect = fiber.return;
  }
}
```

It basically checks if there is layout effect needs to be run on this fiber by `commitLayoutEffectOnFiber()`, and goes to sibling fiber or parent fiber.

```js
function commitLayoutEffectOnFiber(
  finishedRoot: FiberRoot,
  current: Fiber | null,
  finishedWork: Fiber,
  committedLanes: Lanes,
): void {
  if ((finishedWork.flags & LayoutMask) !== NoFlags) {
    switch (finishedWork.tag) {
      case FunctionComponent:
      case ForwardRef:
      case SimpleMemoComponent: {
        if (
          !enableSuspenseLayoutEffectSemantics ||
          !offscreenSubtreeWasHidden
        ) {
          // At this point layout effects have already been destroyed (during mutation phase).
          // This is done to prevent sibling component effects from interfering with each other,
          // e.g. a destroy function in one component should never override a ref set
          // by a create function in another component during the same commit.
          if (
            enableProfilerTimer &&
            enableProfilerCommitHooks &&
            finishedWork.mode & ProfileMode
          ) {
            try {
              startLayoutEffectTimer();
              commitHookEffectListMount(
                HookLayout | HookHasEffect,
                finishedWork,
              );
            } finally {
              recordLayoutEffectDuration(finishedWork);
            }
          } else {
            commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
          }
        }
        break;
      }
  ...
```

We can find `commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork)`, which means

1. `HookLayout` -> this effect is layout effect
2. `HookHasEffect` -> this effect needs to be run

```js
function commitHookEffectListMount(flags: HookFlags, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null =
    (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & flags) === flags) {
        // Mount
        const create = effect.create;
        effect.destroy = create();
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

`commitHookEffectListMount()` is straightforward, it checks against effect tag and run the `create()` function and set up the `destroy`.

## 4. when do layout effects get cleaned up?

We've seen how the cleanup function `destory` is set up, when do they get run? It actually happens before it in `commitMutationEffects()`.

```js
function commitMutationEffectsOnFiber(
  finishedWork: Fiber,
  root: FiberRoot,
  lanes: Lanes,
) {
  const current = finishedWork.alternate;
  const flags = finishedWork.flags;

  // The effect flag should be checked *after* we refine the type of fiber,
  // because the fiber tag is more specific. An exception is any flag related
  // to reconcilation, because those can be set on all fiber types.
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case MemoComponent:
    case SimpleMemoComponent: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      commitReconciliationEffects(finishedWork);

      if (flags & Update) {
        try {
          commitHookEffectListUnmount(
            HookInsertion | HookHasEffect,
            finishedWork,
            finishedWork.return,
          );
          commitHookEffectListMount(
            HookInsertion | HookHasEffect,
            finishedWork,
          );
        } catch (error) {
          captureCommitPhaseError(finishedWork, finishedWork.return, error);
        }
        // Layout effects are destroyed during the mutation phase so that all
        // destroy functions for all fibers are called before any create functions.
        // This prevents sibling component effects from interfering with each other,
        // e.g. a destroy function in one component should never override a ref set
        // by a create function in another component during the same commit.
        if (
          enableProfilerTimer &&
          enableProfilerCommitHooks &&
          finishedWork.mode & ProfileMode
        ) {
          try {
            startLayoutEffectTimer();
            commitHookEffectListUnmount(
              HookLayout | HookHasEffect,
              finishedWork,
              finishedWork.return,
            );
          } catch (error) {
            captureCommitPhaseError(finishedWork, finishedWork.return, error);
          }
          recordLayoutEffectDuration(finishedWork);
        } else {
          try {
            commitHookEffectListUnmount(
              HookLayout | HookHasEffect,
              finishedWork,
              finishedWork.return,
            );
          } catch (error) {
            captureCommitPhaseError(finishedWork, finishedWork.return, error);
          }
        }
      }
      return;
    }
    ...
```

See the comments in the middle, `commitHookEffectListUnmount()` is responsible to run cleanups.

```js
function commitHookEffectListUnmount(
  flags: HookFlags,
  finishedWork: Fiber,
  nearestMountedAncestor: Fiber | null
) {
  const updateQueue: FunctionComponentUpdateQueue | null =
    (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & flags) === flags) {
        // Unmount
        const destroy = effect.destroy;
        effect.destroy = undefined;
        if (destroy !== undefined) {
          safelyCallDestroy(finishedWork, nearestMountedAncestor, destroy);
        }
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

`commitHookEffectListUnmount()` is also straightforward, it get the `destory` function on effect, and then just run it.

Rememeber when deps changes, the effect is set with an effect of `HookHasEffect`.

## 5. but how cleanups get run when component is unmounted?

Good question, other than deps change that triggers the cleanup, component unmount also leads to cleanup, which in fiber world means fiber removal.

Notice there is `recursivelyTraverseMutationEffects()` in above sinppet, actually for cases of fiber deletion, effects are cleaned up there.

```js
function recursivelyTraverseMutationEffects(
  root: FiberRoot,
  parentFiber: Fiber,
  lanes: Lanes
) {
  // Deletions effects can be scheduled on any fiber type. They need to happen
  // before the children effects hae fired.
  const deletions = parentFiber.deletions;
  if (deletions !== null) {
    for (let i = 0; i < deletions.length; i++) {
      const childToDelete = deletions[i];
      try {
        commitDeletionEffects(root, parentFiber, childToDelete);
      } catch (error) {
        captureCommitPhaseError(childToDelete, parentFiber, error);
      }
    }
  }

  const prevDebugFiber = getCurrentDebugFiberInDEV();
  if (parentFiber.subtreeFlags & MutationMask) {
    let child = parentFiber.child;
    while (child !== null) {
      setCurrentDebugFiberInDEV(child);
      commitMutationEffectsOnFiber(child, root, lanes);
      child = child.sibling;
    }
  }
  setCurrentDebugFiberInDEV(prevDebugFiber);
}
```

We can see that it traverse through all the descendent fiber from `fiber.deletions` and run the cleanup functions.

Since deleted fibers are not in the final fiber tree, they are put into their parent fiber for reference.

```js
function deleteChild(returnFiber: Fiber, childToDelete: Fiber): void {
  if (!shouldTrackSideEffects) {
    // Noop.
    return;
  }
  const deletions = returnFiber.deletions;
  if (deletions === null) {
    returnFiber.deletions = [childToDelete];
    returnFiber.flags |= ChildDeletion;
  } else {
    deletions.push(childToDelete);
  }
}
```

And `deletions` are set during reconciliation before commiting.

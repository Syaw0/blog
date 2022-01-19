---
layout: post
title: "The lifecycle of effect hooks in React - React source code walkthrough 16"
date: 2022-01-19 18:21:10 +0900
categories: React
---

> You can watch my [Youtube video explanation](https://www.youtube.com/watch?v=-0-pCZvvwaM) for this post

I've talked about how `useEffect()` works in [this video](https://www.youtube.com/watch?v=Ggmdo7TORNc), but the content was a bit messy. Also the React version I used was not the latest, things have changed sinc then so let me explain the lifecycle of Effect hooks again.

By "Effect hook", I mean "useEffect()", like below.

```js
function A() {
  useEffect(function create() {
    console.log("create effect");
    return function cleanup() {
      console.log("destroy effect");
    };
  }, []);
  return <div />;
}
```

I give the functions names of `create()` and `cleanup()` to differentiate.

Let's answer these 3 questions.

1. what happens when `useEffect()` is first called?
2. what happens when `deps` in `useEffect()` change?
3. when do `cleanup()` get invoked?

# What happens when `useEffect()` is first called ?

`useEffect()` is a way to create Effect hook, a hook is something attached to a fiber.

From [source code](https://github.com/facebook/react/blob/main/packages/react/src/ReactHooks.js#L100-L106), we see that `useEffect()` resolves to [mountEffect](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.old.js#L1698) for first call, and [updateEffect](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.old.js#L1723-L1728) for latter updates.

> Sounds fair, since in first call there is nothing to compare.

Let's see what is in `mountEffect()`.

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

Looks a lot but actually pretty simple, it basically does 2 things:

1. create a new hook in `mountWorkInProgressHook()` which is attached to hook list (`memoizedState`) of the fiber
2. set up an update effect with our creator function, and attach to the `updateQueue` on fiber, and also track the effect in the hook through `memoizedState`, our creator function is not yet invoked.

So a fiber could have:

1. a list up updates (effects), in `updateQueue`
2. a list of hooks in `memoizedState`, for effect hook, it tracks the effects as well.

> I guess we can think of hooks like an internal linked states for a fiber, so `memoizedState` is used.
> Also `useEffect()` pushs an effect in `updateQueue`, so it is called Effect hook.

### Effect.tag is to mark if effect should be run

`Effect` means side effects, put in `updateQueue` on fiber, they will be run after React commits the changes.

See the first argument of `pushEffect`? It is to control `Effect.tag`, in our mounting phase, it is passed with `HookHasEffect | hookFlags`, in **which `HookHasEffect` means it should be run**.

This is very important, we could infer that in `updateEffect`, this flag is going to be toggled by checking if `deps` has changed.

## flushPassiveEffects()

As we've talked in previous videos, [flushPassiveEffects()](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L2150) is the function to run those effects created by `useEffect()`.

It is invoked in a few places, but the most important one is in [commitRoot()](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L1901), which is the commit phase after reconciliation.

```js
if (
  (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
  (finishedWork.flags & PassiveMask) !== NoFlags
) {
  if (!rootDoesHavePassiveEffects) {
    rootDoesHavePassiveEffects = true;
    pendingPassiveEffectsRemainingLanes = remainingLanes;
    scheduleCallback(NormalSchedulerPriority, () => {
      flushPassiveEffects();
      // This render triggered passive effects: release the root cache pool
      // *after* passive effects fire to avoid freeing a cache pool that may
      // be referenced by a node in the tree (HostRoot, Cache boundary etc)
      return null;
    });
  }
}
```

`flushPassiveEffects()` is scheduled by `schedulCallback`, so it is run in next tick, not synchrously right after DOM changes. scheduler is not our topic today, let's look into details of `flushPassiveEffects()`.

> I have another video of [useLayoutEffect()](https://www.youtube.com/watch?v=E7dZM6ZndfA), which is a slight different story.

from the [source code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L2150), we can see that basically id does 2 things.

```js
commitPassiveUnmountEffects(root.current);
commitPassiveMountEffects(root, root.current);
```

An effect's cleanup must be run first before it is run again, so `unmount` happens first before `mount`.

These 2 functions checks effects on all fibers from root, the algorithm is what we [covered in previous post]({% post_url 2022-01-16-fiber-traversal-in-react %})

You might think that wouldn't it be inefficient to traverse through tree over and over again ? You are right, that is why React has this optimization of avoiding unnecessary checks with `finishedWork.subtreeFlags` `finishedWork.flags`, .etc.

## commitPassiveUnmountEffects()

As our algorithm in [last post]({% post_url 2022-01-16-fiber-traversal-in-react %}) explained, `commitPassiveUnmountEffects()` includes a `begin()` and `complete()`.

```js
function commitPassiveUnmountEffects_begin() {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    const child = fiber.child;

    if ((nextEffect.flags & ChildDeletion) !== NoFlags) {
      const deletions = fiber.deletions;
      if (deletions !== null) {
        for (let i = 0; i < deletions.length; i++) {
          const fiberToDelete = deletions[i];
          nextEffect = fiberToDelete;
          commitPassiveUnmountEffectsInsideOfDeletedTree_begin(
            fiberToDelete,
            fiber
          );
        }

        if (deletedTreeCleanUpLevel >= 1) {
          // A fiber was deleted from this parent fiber, but it's still part of
          // the previous (alternate) parent fiber's list of children. Because
          // children are a linked list, an earlier sibling that's still alive
          // will be connected to the deleted fiber via its `alternate`:
          //
          //   live fiber
          //   --alternate--> previous live fiber
          //   --sibling--> deleted fiber
          //
          // We can't disconnect `alternate` on nodes that haven't been deleted
          // yet, but we can disconnect the `sibling` and `child` pointers.
          const previousFiber = fiber.alternate;
          if (previousFiber !== null) {
            let detachedChild = previousFiber.child;
            if (detachedChild !== null) {
              previousFiber.child = null;
              do {
                const detachedSibling = detachedChild.sibling;
                detachedChild.sibling = null;
                detachedChild = detachedSibling;
              } while (detachedChild !== null);
            }
          }
        }

        nextEffect = fiber;
      }
    }

    if ((fiber.subtreeFlags & PassiveMask) !== NoFlags && child !== null) {
      ensureCorrectReturnPointer(child, fiber);
      nextEffect = child;
    } else {
      commitPassiveUnmountEffects_complete();
    }
  }
}
```

It basically does one thing, that is to **cleanup the effects in deleted fibers**.

Why?

Because if some fibers are deleted, it is not on our fiber tree anymore, in order to get reference to them to do some cleanups, React keep track of them on their parent fiber through `deletions`. We'll cover it in details in the future, but anyway, we know how React does it now.

```js
function commitPassiveUnmountEffects_complete() {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    if ((fiber.flags & Passive) !== NoFlags) {
      commitPassiveUnmountOnFiber(fiber);
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
function commitPassiveUnmountOnFiber(finishedWork: Fiber): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent: {
      commitHookEffectListUnmount(
        HookPassive | HookHasEffect,
        finishedWork,
        finishedWork.return
      );
      break;
    }
  }
}
```

In `complete()`, `commitHookEffectListUnmount()` is the core logic, the first argument is `HookPassive | HookHasEffect`, meaning we should rerun **passive effects** which **needs to be run**, as we explained what `effect.tag` does at the beginning.

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

The code is fairly simple, it loop through all the linked effects, check the `tag` to see if matches `HookPassive | HookHasEffect`, and run the `destroy`.

But when we created the effect hook `pushEffect(HookHasEffect | hookFlags, create, undefined,nextDeps)`, we pass in `undefined` as `destroy`.

So when does `destroy` gets set ? I bet you already know it, when we `pushEffect`, it is not mounted yet, `destroy` must be created in `commitPassiveMountEffects()`.

## commitPassiveMountEffects()

The [tree traversal algorithm]({% post_url 2022-01-16-fiber-traversal-in-react %}) is the same, we'll skip it.

Similarly, it triggers `commitHookEffectListMount(HookPassive | HookHasEffect, finishedWork)`.

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

Simple, it set up the destroy by `effect.destroy = create()`, which means our creator function gots run here. Finally!!

Phew, that is a long journey! Stay with us we have 2 more questions to go.

# What happens when `deps` in `useEffect()` change.

After first mount, we use update dispatcher now. If our componet `A()` got run again, `useEffect()` would also be run again, it leads to [updateEffect()](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.old.js#L1723-L1728).

> For when A() gets run again, please refer to my blog post [How does React bailout work in reconciliation]({% post_url 2022-01-08-how-does-bailout-work %})).

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

Things are a bit complex here,

1. what is done in `updateWorkInProgressHook()`?
2. what is `currentHook`?
3. what happens if `areHookInputsEqual()` passes? and what if not.

To begin answering question, we need to recall what we know about reconciliation, basically, React has a fiber tree (`current`) in which each fiber has a `alternate` copy, which means we have a `alternate` fiber tree.

Reconciliation means we doing those updates on the `alternate`, or we say `workInProgress` fiber tree, then just switch to this updated tree.

> See my explanation about reconciliation [in this video](https://www.youtube.com/watch?v=0GM-1W7i9Tk)

Let's first visit 2 global variables, which is used in [updateWorkInProgressHook()](https://github.com/facebook/react/blob/51947a14bb24bd151f76f6fc0acdbbc404de13f7/packages/react-reconciler/src/ReactFiberHooks.old.js#L649),

```js
// Hooks are stored as a linked list on the fiber's memoizedState field. The
// current hook list is the list that belongs to the current fiber. The
// work-in-progress hook list is a new list that will be added to the
// work-in-progress fiber.
let currentHook: Hook | null = null;
let workInProgressHook: Hook | null = null;
```

So we are keeping track of the hook being processed in current tree `currentHook`, and the one in the `workInProgress` tree, which is `workInProgressHook`.

Why we keep track of these two ? because we want to comparing them, so that we know if deps has changed.

Let's see inside of `updateWorkInProgressHook()`.

```js
let nextCurrentHook: null | Hook;
if (currentHook === null) {
  const current = currentlyRenderingFiber.alternate;
  if (current !== null) {
    nextCurrentHook = current.memoizedState;
  } else {
    nextCurrentHook = null;
  }
} else {
  nextCurrentHook = currentHook.next;
}
```

First it updates `nextCurrentHook`, which is the next hook of currentHook in `current tree`. Reasonable since we are going to create the hooks again, we need to find the previous existing hook.

Here we can see why the hooks must be stable, because it all depends on the order.

```js
let nextWorkInProgressHook: null | Hook;
if (workInProgressHook === null) {
  nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
} else {
  nextWorkInProgressHook = workInProgressHook.next;
}
```

And do the same for `nextWorkInProgressHook`.

```js
if (nextWorkInProgressHook !== null) {
  // There's already a work-in-progress. Reuse it.
  workInProgressHook = nextWorkInProgressHook;
  nextWorkInProgressHook = workInProgressHook.next;

  currentHook = nextCurrentHook;
}
```

if we find `nextWorkInProgressHook`, then we can safely move one step forward, by updating `currentHook` and `workInProgressHook`.

```js
else {
  if (nextCurrentHook === null) {
    throw new Error('Rendered more hooks than during the previous render.');
  }

  currentHook = nextCurrentHook;

  const newHook: Hook = {
    memoizedState: currentHook.memoizedState,

    baseState: currentHook.baseState,
    baseQueue: currentHook.baseQueue,
    queue: currentHook.queue,

    next: null,
  };

  if (workInProgressHook === null) {
    // This is the first hook in the list.
    currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
  } else {
    // Append to the end of the list.
    workInProgressHook = workInProgressHook.next = newHook;
  }
}
```

If we don't find `nextWorkInProgressHook`, it means we need to create a new hook.

Why sometimes `nextWorkInProgressHook` is null and sometimes not ?

Good question, when our component `A()` gets rerun in [renderWithHooks()](https://github.com/facebook/react/blob/51947a14bb24bd151f76f6fc0acdbbc404de13f7/packages/react-reconciler/src/ReactFiberHooks.old.js#L366), their `memoizedState` and `updateQueue` are actually reset

```js
workInProgress.memoizedState = null;
workInProgress.updateQueue = null;
workInProgress.lanes = NoLanes;
```

So I guess it always be null, correct me if I'm wrong.

### deps are compared

```js
if (areHookInputsEqual(nextDeps, prevDeps)) {
  hook.memoizedState = pushEffect(hookFlags, create, destroy, nextDeps);
  return;
}
```

If deps are the equal, meaning, we don't need to run this effect hook, just `pushEffect` as if it is mounted, without `hasEffect` flag.

Why not just use the `memoizedState` from current hook? Good question, I am not sure. One thing I know is that `tag` on effect is not reset after effect is done, so we cannot reuse it easily without resetting the tag and cleaning up the `next`, maybe that's why using `pushEffect()` could save effort.

if deps changes, then effect hook needs to be run, which is the last piece of code

```js
hook.memoizedState = pushEffect(
  HookHasEffect | hookFlags,
  create,
  destroy,
  nextDeps
);
```

Now we have `HookHasEffect` flag, which means it is going to be run in `flushPassiveEffects()`.

Cool, seems like our 2nd question has been answered, and also the 3rd one.

# Let's summarize

1. an `Effect` keep tracks our functions passed to `useEffect`, it has following properties
   - `tag`: it might contains `HasEffect` to mark if it needs to run
   - `create`: the function we pass in,
   - `destroy`: the cleanup function returned in `create`
2. `Effect` is linked in a list in order and attached to a fiber in `updateQueue`
3. when `useEffect()` is run for the first time
   - a new hook is created , attached to `memoizedState`
   - a new `Effect` is created with the tag `HasEffect`, attached to the fiber in `updateQueue`
4. when an effect is run (mounted), `create` is invoked and `destroy` is set to the return value, which means
   - if an effect has `HasEffect` flag and also `destroy`, then destroy should be called
5. in `flushPassiveEffects`, it traverse through the fiber tree in 2 passes

   - first search for effects that has `HasEffect` and `destroy`, clean them up
     - also it handles cleanup caused by fiber deletion.
   - then search for effects that has `HasEffect`, and re-run them.

6. when component is rerendered, the workInProgress fiber has empty `memoizedState` and `emptyQueue`. `useEffect()` is run again, `deps` are compared. If changes are found, a new effect is created with `hasEffect`

I'm not sure why the `HasEffect` is not revoked in `flushPassiveEffects()` , maybe because we are only mount/unmount in the commit phase, there is no need to revoke since the reconciliation is already done. The next time we go to render phase, the effects are reconstructed.

Hope it helps you understand the lifecycle of an effect hook. There will be more posts about React, stay tuned.

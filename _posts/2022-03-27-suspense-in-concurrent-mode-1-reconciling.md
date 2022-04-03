---
layout: post
title: "How Suspense works internally in Concurrent Mode 1 - Reconciling flow | React Source Code Walkthrough 23"
date: 2022-04-02 18:21:10 +0900
categories: React
image: /static/reconciling-in-suspense.png
---

I once tried to figure out how Suspense works, you can watch [my youtube video](https://www.youtube.com/watch?v=4Ippewm6AXk), but it was very rough and also not reflecting the latest logic in React 18.

Now I'm going to take a depper look at **how Suspense works in Concurrent mode**. It turned out to be very complex I plan to do it in couple of episodes in following steps

1. reconciling - how Suspense reconciles
2. Offscreen component - the internal component used by Suspense component
3. Suspense context - ??
4. Ping & Retry - make sure rerender nicely after Promises are resolved

Today it is about 1 reconciling.

### Table of contents

- [Suspense Demo](#suspense-demo)
- [Let’s first see how Suspense component renders itself](#lets-first-see-how-suspense-component-renders-itself)
  - [Initial Mount](#initial-mount)
  - [Update](#update)
- [Wrappers inside of Suspense](#wrappers-inside-of-suspense)
  - [mountSuspenseFallbackChildren()](#mountsuspensefallbackchildren)
  - [mountSuspensePrimaryChildren()](#mountsuspenseprimarychildren)
- [How Promise is caught in Suspense and update is triggered?](#how-promise-is-caught-in-suspense-and-update-is-triggered)
  - [throwException()](#throwexception)
  - [What if we don’t find the Suspense?](#what-if-we-dont-find-the-suspense)
  - [completeUnitOfWork()](#completeunitofwork)
- [Summary](#summary)

## Suspense Demo

Open [this very basic Suspense demo](/demos/react/suspense/basic.html).

![](/static/basic-suspense.gif)

The code is very simple, just a basic implementation of throwing Promise when data is not ready.

```jsx
const getResource = (data, delay = 1000) => ({
  _data: null,
  _promise: null,
  status: "pending",
  get data() {
    if (this.status === "ready") {
      return this._data;
    } else {
      if (this._promise == null) {
        this._promise = new Promise((resolve) => {
          setTimeout(() => {
            this._data = data;
            this.status = "ready";
            resolve();
          }, delay);
        });
      }
      throw this._promise;
    }
  },
});

function App() {
  const [resource, setResource] = React.useState(null);
  return (
    <div className="app">
      <button
        onClick={() => {
          setResource(getResource("JSer"));
        }}
      >
        start
      </button>
      <React.Suspense fallback={<p>loading...</p>}>
        <Child resource={resource} />
      </React.Suspense>
    </div>
  );
}
```

Fallback is displayed when the resource is loading, as expected.

## Let's first see how Suspense component renders itself

In `beginWork()` we can find this piece of code. [source](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3925)

```js
case SuspenseComponent:
  return updateSuspenseComponent(current, workInProgress, renderLanes);
```

Meaning both initial render and updates of Suspense are in `updateSuspenseComponent`, it is a huge chunk of code [source](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L2343), let's break it down.

```js
function updateSuspenseComponent(current, workInProgress, renderLanes) {
  const nextProps = workInProgress.pendingProps;

  let suspenseContext: SuspenseContext = suspenseStackCursor.current;

  let showFallback = false;
  const didSuspend = (workInProgress.flags & DidCapture) !== NoFlags;
  if (
    didSuspend ||
    shouldRemainOnFallback(suspenseContext, current, workInProgress, renderLanes)
  ) {
    // Something in this boundary's subtree already suspended. Switch to
    // rendering the fallback children.
    showFallback = true;
    workInProgress.flags &= ~DidCapture;
  }
```

First what is `SuspenseContext`, this is something I'll try to figure it out the upcoming episodes. Let's skip it for now.

`showFallback` is pretty straighforward, it is the variable to determine whether or not show fallback, and it is default to false.

We can see `showFallback` depends on `didSuspend`, in turn depends on `DidCapture`, it is quite important flag, keep it in mind.

`shouldRemainOnFallback()` is something related to Suspense Context, we'll cover it in another episode.

Notice that `DidCapture` is removed so that in futur rerenders we'll try to get the right contents, also meaning the promise would be thrown again. (try [this demo](/demos/react/suspense/rethrow.html))

### Initial mount

```js
if (current === null) {
  const nextPrimaryChildren = nextProps.children;
  const nextFallbackChildren = nextProps.fallback;

  if (showFallback) {
    const fallbackFragment = mountSuspenseFallbackChildren(
      workInProgress,
      nextPrimaryChildren,
      nextFallbackChildren,
      renderLanes
    );
    const primaryChildFragment: Fiber = (workInProgress.child: any);
    primaryChildFragment.memoizedState =
      mountSuspenseOffscreenState(renderLanes);
    workInProgress.memoizedState = SUSPENDED_MARKER;
    return fallbackFragment;
  } else {
    return mountSuspensePrimaryChildren(
      workInProgress,
      nextPrimaryChildren,
      renderLanes
    );
  }
}
```

`current === null` means initial render. `mountSuspenseFallbackChildren()` would mount both primary children(content) and fallback for us, but returns the fallback.

`memoizedState` is also initialzied, this is a marker to indicate that this Suspense is rendering fallback.

If not rendering fallback, `mountSuspensePrimaryChildren()` mount the children for us.

We'll come back to `mountSuspenseFallbackChildren()` and `mountSuspensePrimaryChildren()` later in this episode.

### Update

And for update, the logic is similar actually, depending of current status and status to be, there are four branches, we'll cover it in details

```js
} else {
  // This is an update.
  // If the current fiber has a SuspenseState, that means it's already showing
  // a fallback.
  const prevState: null | SuspenseState = current.memoizedState;
  if (prevState !== null) {
    // The current tree is already showing a fallback
    if (showFallback) {
      // prev: fallback, now: fallback
      ...
    } else {
      // prev: fallback, now: content
      ...
    }
  } else {
    if (showFallback) {
     // prev: content, now: callback
     ...
    } else {
      // prev: content, now: content
      ...
    }
  }
}
```

#### prev: fallback, now: fallback

```js
const nextFallbackChildren = nextProps.fallback;
const nextPrimaryChildren = nextProps.children;
const fallbackChildFragment = updateSuspenseFallbackChildren(
  current,
  workInProgress,
  nextPrimaryChildren,
  nextFallbackChildren,
  renderLanes
);
const primaryChildFragment: Fiber = (workInProgress.child: any);
const prevOffscreenState: OffscreenState | null = (current.child: any)
  .memoizedState;
primaryChildFragment.memoizedState =
  prevOffscreenState === null
    ? mountSuspenseOffscreenState(renderLanes)
    : updateSuspenseOffscreenState(prevOffscreenState, renderLanes);
primaryChildFragment.childLanes = getRemainingWorkInPrimaryTree(
  current,
  renderLanes
);
workInProgress.memoizedState = SUSPENDED_MARKER;
return fallbackChildFragment;
```

Both renders the fallback, but fallback itself might changes. `updateSuspenseFallbackChildren()` does the reconciling.
The part of OffscreenState is a bit confusing, it is related to Suspense Cache, I'll save it for future episode.

#### prev: fallback, now: content

```js
const nextPrimaryChildren = nextProps.children;
const primaryChildFragment = updateSuspensePrimaryChildren(
  current,
  workInProgress,
  nextPrimaryChildren,
  renderLanes
);
workInProgress.memoizedState = null;
return primaryChildFragment;
```

This is simple, just reconcile the children part.

#### prev: content, now: callback

code is similar to `prev: fallback, now: content` skip

#### prev: content, now: content

code is similar to `prev: fallback, now: content`.

## Wrappers inside of Suspense

Suspense component is not some simple component but it wraps children in something like Offscreen component to achieve something nice.

Let's briefly take a look for now and details in the future episodes about Offscreen.

### mountSuspenseFallbackChildren()

Ok, let's see what actually happens in `mountSuspenseFallbackChildren()`.

```js
function mountSuspenseFallbackChildren(
  workInProgress,
  primaryChildren,
  fallbackChildren,
  renderLanes
) {
  const mode = workInProgress.mode;
  const progressedPrimaryFragment: Fiber | null = workInProgress.child;

  const primaryChildProps: OffscreenProps = {
    mode: "hidden",
    children: primaryChildren,
  };

  let primaryChildFragment;
  let fallbackChildFragment;

  primaryChildFragment = mountWorkInProgressOffscreenFiber(
    primaryChildProps,
    mode,
    NoLanes
  );
  fallbackChildFragment = createFiberFromFragment(
    fallbackChildren,
    mode,
    renderLanes,
    null
  );

  primaryChildFragment.return = workInProgress;
  fallbackChildFragment.return = workInProgress;
  primaryChildFragment.sibling = fallbackChildFragment;
  workInProgress.child = primaryChildFragment;
  return fallbackChildFragment;
}
```

1. primary child is wrapped in Offsscreen Fiber, mode set to `hidden`
2. fallback is wrapped in Fragment.
3. primary child and fallback are both placed as children.

Why wrapps fallback in Fragment?

I guess because fallback is type of `ReactNodeList` and it could be number or string and normally string would requires us to do special handling, like [here](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L534), so wrapping it in Fragment look easier to handle.

```js
export type ReactNode =
  | React$Element<any>
  | ReactPortal
  | ReactText
  | ReactFragment
  | ReactProvider<any>
  | ReactConsumer<any>;

export type ReactEmpty = null | void | boolean;

export type ReactFragment = ReactEmpty | Iterable<React$Node>;

export type ReactNodeList = ReactEmpty | React$Node;

export type ReactText = string | number;
```

OK, here is a diagram about the fiber structure of Suspense.

![](/static/suspense-fiber-structure-hidden.png)

What is special about `mountWorkInProgressOffscreenFiber`?

```js
function mountWorkInProgressOffscreenFiber(
  offscreenProps: OffscreenProps,
  mode: TypeOfMode,
  renderLanes: Lanes
) {
  // The props argument to `createFiberFromOffscreen` is `any` typed, so we use
  // this wrapper function to constrain it.
  return createFiberFromOffscreen(offscreenProps, mode, NoLanes, null);
}
export function createFiberFromOffscreen(
  pendingProps: OffscreenProps,
  mode: TypeOfMode,
  lanes: Lanes,
  key: null | string
) {
  const fiber = createFiber(OffscreenComponent, pendingProps, key, mode);
  fiber.elementType = REACT_OFFSCREEN_TYPE;
  fiber.lanes = lanes;
  const primaryChildInstance: OffscreenInstance = {};
  fiber.stateNode = primaryChildInstance;
  return fiber;
}
```

Nothing fancy, but it has `mode` property to indicate if it is `hidden` or `visible`.

### mountSuspensePrimaryChildren()

```js
function mountSuspensePrimaryChildren(
  workInProgress,
  primaryChildren,
  renderLanes
) {
  const mode = workInProgress.mode;
  const primaryChildProps: OffscreenProps = {
    mode: "visible",
    children: primaryChildren,
  };
  const primaryChildFragment = mountWorkInProgressOffscreenFiber(
    primaryChildProps,
    mode,
    renderLanes
  );
  primaryChildFragment.return = workInProgress;
  workInProgress.child = primaryChildFragment;
  return primaryChildFragment;
}
```

Here again we are using Offscreen fiber, but this time without fallback and mode is "visible".

![](/static/suspense-fiber-structure-visible.png)

Btw, `workInProgress` also has `mode`, but a different one - `TypeOfMode`.

```js
export type TypeOfMode = number;

export const NoMode = /*                         */ 0b000000;
// TODO: Remove ConcurrentMode by reading from the root tag instead
export const ConcurrentMode = /*                 */ 0b000001;
export const ProfileMode = /*                    */ 0b000010;
export const DebugTracingMode = /*               */ 0b000100;
export const StrictLegacyMode = /*               */ 0b001000;
export const StrictEffectsMode = /*              */ 0b010000;
export const ConcurrentUpdatesByDefaultMode = /* */ 0b100000;
```

You might wonder why we keep the primary children in the fiber tree? Why not just remove them? It is a greate question, simply speaking it is to keep the state of the fibers, we don't want everything to be fresh new after switching back from fallback. Details are to come in future episode of Offscreen.

Now let's find out how Promise comes in play.

## How Promise is caught in Suspense and update is triggered?

We've already know that suspense reacts when a promise is thrown, which is part of the error handling, so let's go to `handleError` first. ([source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1492))

```js
function handleError(root, thrownValue): void {
  do {
    let erroredWork = workInProgress;
    try {
      // Reset module-level state that was set during the render phase.
      resetContextDependencies();
      resetHooksAfterThrow();
      // TODO: I found and added this missing line while investigating a
      // separate issue. Write a regression test using string refs.
      ReactCurrentOwner.current = null;

      throwException(
        root,
        erroredWork.return,
        erroredWork,
        thrownValue,
        workInProgressRootRenderLanes
      );
      completeUnitOfWork(erroredWork);
    } catch (yetAnotherThrownValue) {
      // Something in the return path also threw.
      thrownValue = yetAnotherThrownValue;
      if (workInProgress === erroredWork && erroredWork !== null) {
        // If this boundary has already errored, then we had trouble processing
        // the error. Bubble it to the next boundary.
        erroredWork = erroredWork.return;
        workInProgress = erroredWork;
      } else {
        erroredWork = workInProgress;
      }
      continue;
    }
    // Return to the normal work loop.
    return;
  } while (true);
}
```

So the key parts are these 2 function calls

1. `throwException`
2. `completeUnitOfWork`

### throwException

[source](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberThrow.new.js#L430)

It is a huge chunk of code, let's break it down. First the throwing fiber is marked as Incomplete.

```js
// The source fiber did not complete.
sourceFiber.flags |= Incomplete;
```

Then it checks if the error is thenable, if it is then the component suspends.

```js
if (
  value !== null &&
  typeof value === 'object' &&
  typeof value.then === 'function'
) {
  // This is a wakeable. The component suspended.
  const wakeable: Wakeable = (value: any);
  ...
} else {
  // regular error
}
```

We can think of `wakeable` as just Promise that is thrown. If not Promise, then it is just normal error which should be handled by Error Boundary (watch [my video for ErrorBoundary](https://www.youtube.com/watch?v=0TnuJKLjMyg))

Now let's focus on the Suspense branch.

```js
// Schedule the nearest Suspense to re-render the timed out view.
const suspenseBoundary = getNearestSuspenseBoundaryToCapture(returnFiber);
```

It first get the nearest Suspense. It is call Suspense Boundary here we can see it is pretty similar to Error Boundary.

`getNearestSuspenseBoundaryToCapture` should be simple and we'll skip, it just recursively trace back on the ancestor fiber nodes by looking at `return`. [source](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberThrow.new.js#L277).

```js
if (suspenseBoundary !== null) {
  suspenseBoundary.flags &= ~ForceClientRender;
  markSuspenseBoundaryShouldCapture(
    suspenseBoundary,
    returnFiber,
    sourceFiber,
    root,
    rootRenderLanes
  );
  // We only attach ping listeners in concurrent mode. Legacy Suspense always
  // commits fallbacks synchronously, so there are no pings.
  if (suspenseBoundary.mode & ConcurrentMode) {
    attachPingListener(root, wakeable, rootRenderLanes);
  }
  attachRetryListener(suspenseBoundary, root, wakeable, rootRenderLanes);
  return;
}
```

After we find the Suspense Boundary, we do 3 things here

1. `markSuspenseBoundaryShouldCapture()`
2. `attachPingListener()`
3. `attachRetryListener()`

Obviously, `markSuspenseBoundaryShouldCapture()` is for Suspense to render fallbacks, and the other 2 are somehow attaching callbacks to the promise, because when they are settled, we need to render the contents.

2 & 3 will be explained in details in future episode of Ping & Retry.

### What if we don't find the Suspense.

We can continue the code and know that if it is not SyncLane, then it is fine to have no Suspense Boundary.

```js
else {
  // No boundary was found. Unless this is a sync update, this is OK.
  // We can suspend and wait for more data to arrive.

  if (!includesSyncLane(rootRenderLanes)) {
    // This is not a sync update. Suspend. Since we're not activating a
    // Suspense boundary, this will unwind all the way to the root without
    // performing a second pass to render a fallback. (This is arguably how
    // refresh transitions should work, too, since we're not going to commit
    // the fallbacks anyway.)
    //
    // This case also applies to initial hydration.
    attachPingListener(root, wakeable, rootRenderLanes);
    renderDidSuspendDelayIfPossible();
    return;
  }

  // This is a sync/discrete update. We treat this case like an error
  // because discrete renders are expected to produce a complete tree
  // synchronously to maintain consistency with external state.
  const uncaughtSuspenseError = new Error(
    "A component suspended while responding to synchronous input. This " +
      "will cause the UI to be replaced with a loading indicator. To " +
      "fix, updates that suspend should be wrapped " +
      "with startTransition."
  );

  // If we're outside a transition, fall through to the regular error path.
  // The error will be caught by the nearest suspense boundary.
  value = uncaughtSuspenseError;
}
```

Simply put, if the suspense is caused by user action, then suspense boundary needs to be there.

If not user action or in transition, then `attachPingListener()` and `renderDidSuspendDelayIfPossible()` will try to recover.

Here is [a demo of using transition but without Suspense boundary](/demos/react/suspense/transition.html), we can see it still works.

### markSuspenseBoundaryShouldCapture

[source](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberThrow.new.js#L297)

In `markSuspenseBoundaryShouldCapture()` it handles Legacy Suspense which is used before Concurrent mode, which I guess is the version I happened to meet before so let's ignore it, focusing on Concurrent Mode only.

```js
suspenseBoundary.flags |= ShouldCapture;
```

`ShouldCapture` is set here, there must be some step it is converted to `DidCapture`, we'll come to it hang on.

```js
sourceFiber.flags |= ForceUpdateForLegacySuspense;
// We're going to commit this fiber even though it didn't complete.
// But we shouldn't call any lifecycle methods or callbacks. Remove
// all lifecycle effect tags.
sourceFiber.flags &= ~(LifecycleEffectMask | Incomplete);
```

For source fiber we've already marked ias Incomplete, but here the flags are removed.

```js
export const LifecycleEffectMask =
  Passive | Update | Callback | Ref | Snapshot | StoreConsistency;
```

`LifecycleEffectMask` includes all the side effects, so this means that

**We are treating it as fake complete without really complete**.

```js
// The source fiber did not complete. Mark it with Sync priority to
// indicate that it still has pending work.
sourceFiber.lanes = mergeLanes(sourceFiber.lanes, SyncLane);
```

This relates to the removal of DidCapture when Suspense renders. When we rerenders, we want to make sure the errored component is rendered again, so `lanes` is be set to avoid [bailout]({% post_url 2022-01-08-how-does-bailout-work %}).

Then we go to `completeUnitOfWork(erroredWork)`.

# completeUnitOfWork

After throwException() is done. `completeUnitOfWork()` is called. ([source](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1858))

Since in suspense, the work is Incomplete, we'll only look at the Incomplete branch

```js
function completeUnitOfWork(unitOfWork: Fiber): void {
  // Attempt to complete the current unit of work, then move to the next
  // sibling. If there are no more siblings, return to the parent fiber.
  let completedWork = unitOfWork;
  do {
    // The current, flushed, state of this fiber is the alternate. Ideally
    // nothing should rely on this, but relying on it here means that we don't
    // need an additional field on the work in progress.
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;

    // Check if the work completed or if something threw.
    if ((completedWork.flags & Incomplete) === NoFlags) {
      ...
    } else {
      // This fiber did not complete because something threw. Pop values off
      // the stack without entering the complete phase. If this is a boundary,
      // capture values if possible.
      const next = unwindWork(current, completedWork, subtreeRenderLanes);

      // Because this fiber did not complete, don't reset its lanes.

      if (next !== null) {
        // If completing this work spawned new work, do that next. We'll come
        // back here again.
        // Since we're restarting, remove anything that is not a host effect
        // from the effect tag.
        next.flags &= HostEffectMask;
        workInProgress = next;
        return;
      }

      if (returnFiber !== null) {
        // Mark the parent fiber as incomplete and clear its subtree flags.
        returnFiber.flags |= Incomplete;
        returnFiber.subtreeFlags = NoFlags;
        returnFiber.deletions = null;
      } else {
        // We've unwound all the way to the root.
        workInProgressRootExitStatus = RootDidNotComplete;
        workInProgress = null;
        return;
      }
    }

    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      workInProgress = siblingFiber;
      return;
    }
    // Otherwise, return to the parent
    completedWork = returnFiber;
    // Update the next thing we're working on in case something throws.
    workInProgress = completedWork;
  } while (completedWork !== null);

  // We've reached the root.
  if (workInProgressRootExitStatus === RootInProgress) {
    workInProgressRootExitStatus = RootCompleted;
  }
}
```

As explained in the [my post of traversal algorithm]({% post_url 2022-01-16-fiber-traversal-in-react %}), completeUnitWork is the last step in reconciling a fiber node.

For the incomplete fiber node

```js
const next = unwindWork(current, completedWork, subtreeRenderLanes);
// Because this fiber did not complete, don't reset its lanes.
if (next !== null) {
  // If completing this work spawned new work, do that next. We'll come
  // back here again.
  // Since we're restarting, remove anything that is not a host effect
  // from the effect tag.
  next.flags &= HostEffectMask;
  workInProgress = next;
  return;
}
```

We can see that it offers a chance to continue some work if it is returned from unwindWork.

Also it will recursively mark ancestor node to incomplete.

Per its name, `unwindWork` does some clean up for context .etc. [Source](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberUnwindWork.new.js#L52)

```js
case SuspenseComponent: {
  popSuspenseContext(workInProgress);
  const flags = workInProgress.flags;
  if (flags & ShouldCapture) {
    workInProgress.flags = (flags & ~ShouldCapture) | DidCapture;
    // Captured a suspense effect. Re-render the boundary.
    if (
      enableProfilerTimer &&
      (workInProgress.mode & ProfileMode) !== NoMode
    ) {
      transferActualDuration(workInProgress);
    }
    return workInProgress;
  }
  return null;
}
```

When it unwinds to Suspense, we can see that it

1. pop suspense context, we'll cover it in future episodes
2. if find `ShouldCapture`, then set it do `DidCapture` and returns itself.

Yeah, `ShouldCapture` is converted to `DidCapture` in the complete phase.

## Summary

What a long journey lets's summarize.

1. Suspense use a flag `DidCapture` to decide what to render fallback or contents (primary children)
2. Suspense wraps contents in Offscreen component, so that even when fallback is rendered, contents are not removed from fiber tree, this is to keep the state inside.
3. During reconciling, Suspense decides to skip Offscreen or not based the flag `DidCapture`, this creates the effect of "hidding some fibers"
4. When a promise is thrown
   - nearest Suspense boundary is found and flag is set with `ShouldCapture`, promises are chained with ping & retry listeners
   - since errored, start to complete work, all fibers from errored components up to Suspense will be completed as Incomplete
   - when try to complete nearest Suspense, `ShouldCapture` is marked as `DidCapture` and returns Suspense itself
   - workloop continues reconciling Suspense, this time, rendering fallback branch
5. When a promise is resolved
   - ping & retry listeners make sure rerender happens. (more details in future episodes)

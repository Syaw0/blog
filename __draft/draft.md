---
layout: post
title: "Suspense and Expiration of lanes - React Source Code Walkthrough 22"
date: 2022-03-27 18:21:10 +0900
categories: React
image: /static/21-lanes.png
---

In [preview post]({% post_url 2022-03-26-lanes-in-react %}), we've seen how React internals uses the Lane model to prioritize update on unit of fiber node.

Also in [How React Scheduler works?]({% post_url 2022-03-16-how-react-scheduler-works %}), we've understood how scheduler schedules different tasks based on expiration.

So here is the thing which seems natural, we have lane-level suspending and expiration.

# First let's look at Lane expiration

# There are for root properties

```js
const pendingLanes = root.pendingLanes;
const suspendedLanes = root.suspendedLanes;
const pingedLanes = root.pingedLanes;
const expirationTimes = root.expirationTimes;
```

# pendingLanes

Ok, when a fiber has some update `scheduleUpdateOnFiber()` is called, inside of it, there is this line

```js
// Mark that the root has a pending update.
markRootUpdated(root, lane, eventTime);
```

[markRootUpdated](https://github.com/facebook/react/blob/adb8ebc927ea091ba5ffba6a9f30dbe62eaee0c5/packages/react-reconciler/src/ReactFiberLane.old.js#L573)

```js
export function markRootUpdated(
  root: FiberRoot,
  updateLane: Lane,
  eventTime: number
) {
  root.pendingLanes |= updateLane;

  // If there are any suspended transitions, it's possible this new update
  // could unblock them. Clear the suspended lanes so that we can try rendering
  // them again.
  //
  // TODO: We really only need to unsuspend only lanes that are in the
  // `subtreeLanes` of the updated fiber, or the update lanes of the return
  // path. This would exclude suspended updates in an unrelated sibling tree,
  // since there's no way for this update to unblock it.
  //
  // We don't do this if the incoming update is idle, because we never process
  // idle updates until after all the regular updates have finished; there's no
  // way it could unblock a transition.
  if (updateLane !== IdleLane) {
    root.suspendedLanes = NoLanes;
    root.pingedLanes = NoLanes;
  }

  const eventTimes = root.eventTimes;
  const index = laneToIndex(updateLane);
  // We can always overwrite an existing timestamp because we prefer the most
  // recent event, and we assume time is monotonically increasing.
  eventTimes[index] = eventTime;
}
```

Looks like `markRootUpdated` is to set up some variables on root for easier access. The first line is simple, `pendingLanes` just means the lanes that have work to do.

```js
root.pendingLanes |= updateLane;
```

## What is eventTimes?

Seems like to track the start time of lanes.

```js
eventTimes[index] = eventTime;
```

```js
function pickArbitraryLaneIndex(lanes: Lanes) {
  return 31 - clz32(lanes);
}

function laneToIndex(lane: Lane) {
  return pickArbitraryLaneIndex(lane);
}
```

Ok, so the index is actually counted from right to left. This means `eventTimes` is actually an array of 31 to track the event time which lane is scheduled.

Seems like the time is always overwritten ? Is thi problematic?

## expirationTimes

in `ensureRootIsScheduled()`, we can find below line of code at the very beginning

```js
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  const existingCallbackNode = root.callbackNode;

  // Check if any lanes are being starved by other work. If so, mark them as
  // expired so we know to work on those next.
  markStarvedLanesAsExpired(root, currentTime);
  ...
}
```

Per its name, `markStarvedLanesAsExpired()` is to mark lanes as expired.

```js
export function markStarvedLanesAsExpired(
  root: FiberRoot,
  currentTime: number
): void {
  // TODO: This gets called every time we yield. We can optimize by storing
  // the earliest expiration time on the root. Then use that to quickly bail out
  // of this function.

  const pendingLanes = root.pendingLanes;
  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;
  const expirationTimes = root.expirationTimes;

  // Iterate through the pending lanes and check if we've reached their
  // expiration time. If so, we'll assume the update is being starved and mark
  // it as expired to force it to finish.
  let lanes = pendingLanes;
  while (lanes > 0) {
    const index = pickArbitraryLaneIndex(lanes);
    const lane = 1 << index;

    const expirationTime = expirationTimes[index];
    if (expirationTime === NoTimestamp) {
      // Found a pending lane with no expiration time. If it's not suspended, or
      // if it's pinged, assume it's CPU-bound. Compute a new expiration time
      // using the current time.
      if (
        (lane & suspendedLanes) === NoLanes ||
        (lane & pingedLanes) !== NoLanes
      ) {
        // Assumes timestamps are monotonically increasing.
        expirationTimes[index] = computeExpirationTime(lane, currentTime);
      }
    } else if (expirationTime <= currentTime) {
      // This lane expired
      root.expiredLanes |= lane;
    }

    lanes &= ~lane;
  }
}
```

So `expirationTimes` is like `eventTimes` an array of 31, we can see that it loop through all the lanes

1. if there is no expiration time, set it with an new expiration
2. if there is expriation time and already expired, add it to `expiredLanes`.

# How expiredLanes is treated?

it is only used in `includesExpiredLane()`, and used in deciding if concurrent mode should fallback to sync mode. [source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L901)

```js
const shouldTimeSlice =
  !includesBlockingLane(root, lanes) &&
  !includesExpiredLane(root, lanes) &&
  (disableSchedulerTimeoutInWorkLoop || !didTimeout);
```

# Suspense in Lanes

Here is where `suspendedLanes` is set

```js
function markRootSuspended(root, suspendedLanes) {
  // When suspending, we should always exclude lanes that were pinged or (more
  // rarely, since we try to avoid it) updated during the render phase.
  // TODO: Lol maybe there's a better way to factor this besides this
  // obnoxiously named function :)
  suspendedLanes = removeLanes(suspendedLanes, workInProgressRootPingedLanes);
  suspendedLanes = removeLanes(
    suspendedLanes,
    workInProgressRootInterleavedUpdatedLanes
  );
  markRootSuspended_dontCallThisOneDirectly(root, suspendedLanes);
}

export function markRootSuspended(root: FiberRoot, suspendedLanes: Lanes) {
  root.suspendedLanes |= suspendedLanes;
  root.pingedLanes &= ~suspendedLanes;

  // The suspended lanes are no longer CPU-bound. Clear their expiration times.
  const expirationTimes = root.expirationTimes;
  let lanes = suspendedLanes;
  while (lanes > 0) {
    const index = pickArbitraryLaneIndex(lanes);
    const lane = 1 << index;

    expirationTimes[index] = NoTimestamp;

    lanes &= ~lane;
  }
}
```

it sets `suspendedLanes` and clears the expriation time on them since they are suspended, no need to flush the work on them

When is `markRootSuspended()` called?

# markRootSuspended

it is used in quite a few places but mainly in 2

```js
if (exitStatus === RootFatalErrored) {
  const fatalError = workInProgressRootFatalError;
  prepareFreshStack(root, NoLanes);
  markRootSuspended(root, lanes);
  ensureRootIsScheduled(root, now());
  throw fatalError;
}
```

also in `finishConcurrentRender`

```js
function finishConcurrentRender(root, exitStatus, lanes) {
  switch (exitStatus) {
    case RootIncomplete:
    case RootFatalErrored: {
      throw new Error('Root did not complete. This is a bug in React.');
    }
    // Flow knows about invariant, so it complains if I add a break
    // statement, but eslint doesn't know about invariant, so it complains
    // if I do. eslint-disable-next-line no-fallthrough
    case RootErrored: {
      // We should have already attempted to retry this tree. If we reached
      // this point, it errored again. Commit it.
      commitRoot(root);
      break;
    }
    case RootSuspended: {
      markRootSuspended(root, lanes);
      ...
    }
    case RootSuspendedWithDelay: {
      markRootSuspended(root, lanes);
      ...
```

What is `RootFatalErrored`?

```js
const RootIncomplete = 0;
const RootFatalErrored = 1;
const RootErrored = 2;
const RootSuspended = 3;
const RootSuspendedWithDelay = 4;
const RootCompleted = 5;
```

There are 6 of them

1. `RootIncomplete`, as we mentioned in previous post, this is means it yielded
2. `RootFatalErrored`

set [here](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L1492)

It is set in `handleError()`, meaning the error is not handled.

3. `RootErrored`

it is set in `renderDidError()`. which is from `throwException()` [source](https://github.com/facebook/react/blob/adb8ebc927ea091ba5ffba6a9f30dbe62eaee0c5/packages/react-reconciler/src/ReactFiberThrow.new.js#L551)

throwException is actually called in handleError? Aha, when error happens on root, it is Fatal error,
for non-root node, throwException called.

```js
// We didn't find a boundary that could handle this type of exception. Start
// over and traverse parent path again, this time treating the exception
// as an error.
renderDidError(value);
```

```js
export function renderDidError(error: mixed) {
  if (workInProgressRootExitStatus !== RootSuspendedWithDelay) {
    workInProgressRootExitStatus = RootErrored;
  }
  if (workInProgressRootConcurrentErrors === null) {
    workInProgressRootConcurrentErrors = [error];
  } else {
    workInProgressRootConcurrentErrors.push(error);
  }
}
```

We see that it set exit status to RootErrored when previous is not RootSuspendedWithDelay.

4. `RootSuspended`

```js
export function renderDidSuspend(): void {
  if (workInProgressRootExitStatus === RootIncomplete) {
    workInProgressRootExitStatus = RootSuspended;
  }
}
```

it is only called in `completeWork()` of `SuspenseComponent`.

seems like we need to figure out `Offscreen` and `Suspense` first.

```js
if (
  hasInvisibleChildContext ||
  hasSuspenseContext(
    suspenseStackCursor.current,
    (InvisibleParentSuspenseContext: SuspenseContext)
  )
) {
  // If this was in an invisible tree or a new render, then showing
  // this boundary is ok.
  renderDidSuspend();
} else {
  // Otherwise, we're going to have to hide content so we should
  // suspend for longer if possible.
  renderDidSuspendDelayIfPossible();
}
```

No clue what it is

Seems like `RootSuspended` is for the initial suspend, while `RootSuspendedWithDelay` is error in suspense,

RootSuspendedWithDelay is for transition?

5. `RootSuspendedWithDelay`.

```js
export function renderDidSuspendDelayIfPossible(): void {
  if (
    workInProgressRootExitStatus === RootIncomplete ||
    workInProgressRootExitStatus === RootSuspended ||
    workInProgressRootExitStatus === RootErrored
  ) {
    workInProgressRootExitStatus = RootSuspendedWithDelay;
  }

  // Check if there are updates that we skipped tree that might have unblocked
  // this render.
  if (
    workInProgressRoot !== null &&
    (includesNonIdleWork(workInProgressRootSkippedLanes) ||
      includesNonIdleWork(workInProgressRootInterleavedUpdatedLanes))
  ) {
    // Mark the current render as suspended so that we switch to working on
    // the updates that were skipped. Usually we only suspend at the end of
    // the render phase.
    // TODO: We should probably always mark the root as suspended immediately
    // (inside this function), since by suspending at the end of the render
    // phase introduces a potential mistake where we suspend lanes that were
    // pinged or updated while we were rendering.
    markRootSuspended(workInProgressRoot, workInProgressRootRenderLanes);
  }
}
```

No clues what what this is

6. `RootCompleted`, yeah, completed.

Let's see the difference between `RootSuspended` and `RootSuspendedWithDelay`.

# Lanes in Suspense

I've briefly touch [How Suspense works](https://www.youtube.com/watch?v=4Ippewm6AXk) in my youtube channel.

Let's dive into some details of it with Lanes.

As we all know, Suspense is triggered by throwing a promise, the logic is in `throwException()` [source](https://github.com/facebook/react/blob/adb8ebc927ea091ba5ffba6a9f30dbe62eaee0c5/packages/react-reconciler/src/ReactFiberThrow.old.js#L430).

```js
if (
  value !== null &&
  typeof value === "object" &&
  typeof value.then === "function"
) {
  // This is a wakeable. The component suspended.
  const wakeable: Wakeable = (value: any);
  resetSuspendedComponent(sourceFiber, rootRenderLanes);

  // Schedule the nearest Suspense to re-render the timed out view.
  const suspenseBoundary = getNearestSuspenseBoundaryToCapture(returnFiber);
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
  } else {
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
}
```

Let's break it down.

1. if the `error` is thennable, then try to suspend.
2. like Error boundary, the nearest Suspense is fetched through getNearestSuspenseBoundaryToCapture
3. `markSuspenseBoundaryShouldCapture`
4. `attachPingListener`
5. `attachRetryListener`

```js
function markSuspenseBoundaryShouldCapture(
  suspenseBoundary: Fiber,
  returnFiber: Fiber,
  sourceFiber: Fiber,
  root: FiberRoot,
  rootRenderLanes: Lanes
): Fiber | null {
  // This marks a Suspense boundary so that when we're unwinding the stack,
  // it captures the suspended "exception" and does a second (fallback) pass.
  if ((suspenseBoundary.mode & ConcurrentMode) === NoMode) {
    // Legacy Mode Suspense
    //
    // If the boundary is in legacy mode, we should *not*
    // suspend the commit. Pretend as if the suspended component rendered
    // null and keep rendering. When the Suspense boundary completes,
    // we'll do a second pass to render the fallback.
    if (suspenseBoundary === returnFiber) {
      // Special case where we suspended while reconciling the children of
      // a Suspense boundary's inner Offscreen wrapper fiber. This happens
      // when a React.lazy component is a direct child of a
      // Suspense boundary.
      //
      // Suspense boundaries are implemented as multiple fibers, but they
      // are a single conceptual unit. The legacy mode behavior where we
      // pretend the suspended fiber committed as `null` won't work,
      // because in this case the "suspended" fiber is the inner
      // Offscreen wrapper.
      //
      // Because the contents of the boundary haven't started rendering
      // yet (i.e. nothing in the tree has partially rendered) we can
      // switch to the regular, concurrent mode behavior: mark the
      // boundary with ShouldCapture and enter the unwind phase.
      suspenseBoundary.flags |= ShouldCapture;
    } else {
      suspenseBoundary.flags |= DidCapture;
      sourceFiber.flags |= ForceUpdateForLegacySuspense;

      // We're going to commit this fiber even though it didn't complete.
      // But we shouldn't call any lifecycle methods or callbacks. Remove
      // all lifecycle effect tags.
      sourceFiber.flags &= ~(LifecycleEffectMask | Incomplete);

      if (supportsPersistence && enablePersistentOffscreenHostContainer) {
        // Another legacy Suspense quirk. In persistent mode, if this is the
        // initial mount, override the props of the host container to hide
        // its contents.
        const currentSuspenseBoundary = suspenseBoundary.alternate;
        if (currentSuspenseBoundary === null) {
          const offscreenFiber: Fiber = (suspenseBoundary.child: any);
          const offscreenContainer = offscreenFiber.child;
          if (offscreenContainer !== null) {
            const children = offscreenContainer.memoizedProps.children;
            const containerProps = getOffscreenContainerProps(
              "hidden",
              children
            );
            offscreenContainer.pendingProps = containerProps;
            offscreenContainer.memoizedProps = containerProps;
          }
        }
      }

      if (sourceFiber.tag === ClassComponent) {
        const currentSourceFiber = sourceFiber.alternate;
        if (currentSourceFiber === null) {
          // This is a new mount. Change the tag so it's not mistaken for a
          // completed class component. For example, we should not call
          // componentWillUnmount if it is deleted.
          sourceFiber.tag = IncompleteClassComponent;
        } else {
          // When we try rendering again, we should not reuse the current fiber,
          // since it's known to be in an inconsistent state. Use a force update to
          // prevent a bail out.
          const update = createUpdate(NoTimestamp, SyncLane);
          update.tag = ForceUpdate;
          enqueueUpdate(sourceFiber, update, SyncLane);
        }
      }

      // The source fiber did not complete. Mark it with Sync priority to
      // indicate that it still has pending work.
      sourceFiber.lanes = mergeLanes(sourceFiber.lanes, SyncLane);
    }
    return suspenseBoundary;
  }
  // Confirmed that the boundary is in a concurrent mode tree. Continue
  // with the normal suspend path.
  //
  // After this we'll use a set of heuristics to determine whether this
  // render pass will run to completion or restart or "suspend" the commit.
  // The actual logic for this is spread out in different places.
  //
  // This first principle is that if we're going to suspend when we complete
  // a root, then we should also restart if we get an update or ping that
  // might unsuspend it, and vice versa. The only reason to suspend is
  // because you think you might want to restart before committing. However,
  // it doesn't make sense to restart only while in the period we're suspended.
  //
  // Restarting too aggressively is also not good because it starves out any
  // intermediate loading state. So we use heuristics to determine when.

  // Suspense Heuristics
  //
  // If nothing threw a Promise or all the same fallbacks are already showing,
  // then don't suspend/restart.
  //
  // If this is an initial render of a new tree of Suspense boundaries and
  // those trigger a fallback, then don't suspend/restart. We want to ensure
  // that we can show the initial loading state as quickly as possible.
  //
  // If we hit a "Delayed" case, such as when we'd switch from content back into
  // a fallback, then we should always suspend/restart. Transitions apply
  // to this case. If none is defined, JND is used instead.
  //
  // If we're already showing a fallback and it gets "retried", allowing us to show
  // another level, but there's still an inner boundary that would show a fallback,
  // then we suspend/restart for 500ms since the last time we showed a fallback
  // anywhere in the tree. This effectively throttles progressive loading into a
  // consistent train of commits. This also gives us an opportunity to restart to
  // get to the completed state slightly earlier.
  //
  // If there's ambiguity due to batching it's resolved in preference of:
  // 1) "delayed", 2) "initial render", 3) "retry".
  //
  // We want to ensure that a "busy" state doesn't get force committed. We want to
  // ensure that new initial loading states can commit as soon as possible.
  suspenseBoundary.flags |= ShouldCapture;
  // TODO: I think we can remove this, since we now use `DidCapture` in
  // the begin phase to prevent an early bailout.
  suspenseBoundary.lanes = rootRenderLanes;
  return suspenseBoundary;
}
```

Phew, a super long piece of comments.

`attachRetryListener` is the promise then callback, it has low priority so RetryLanes.

```js
function attachPingListener(root: FiberRoot, wakeable: Wakeable, lanes: Lanes) {
  // Attach a ping listener
  //
  // The data might resolve before we have a chance to commit the fallback. Or,
  // in the case of a refresh, we'll never commit a fallback. So we need to
  // attach a listener now. When it resolves ("pings"), we can decide whether to
  // try rendering the tree again.
  //
  // Only attach a listener if one does not already exist for the lanes
  // we're currently rendering (which acts like a "thread ID" here).
  //
  // We only need to do this in concurrent mode. Legacy Suspense always
  // commits fallbacks synchronously, so there are no pings.
  let pingCache = root.pingCache;
  let threadIDs;
  if (pingCache === null) {
    pingCache = root.pingCache = new PossiblyWeakMap();
    threadIDs = new Set();
    pingCache.set(wakeable, threadIDs);
  } else {
    threadIDs = pingCache.get(wakeable);
    if (threadIDs === undefined) {
      threadIDs = new Set();
      pingCache.set(wakeable, threadIDs);
    }
  }
  if (!threadIDs.has(lanes)) {
    // Memoize using the thread ID to prevent redundant listeners.
    threadIDs.add(lanes);
    const ping = pingSuspendedRoot.bind(null, root, wakeable, lanes);
    if (enableUpdaterTracking) {
      if (isDevToolsPresent) {
        // If we have pending work still, restore the original updaters
        restorePendingUpdaters(root, lanes);
      }
    }
    wakeable.then(ping, ping);
  }
}
```

let's see what is in `pingSuspendedRoot()` ?

```js
export function pingSuspendedRoot(
  root: FiberRoot,
  wakeable: Wakeable,
  pingedLanes: Lanes
) {
  const pingCache = root.pingCache;
  if (pingCache !== null) {
    // The wakeable resolved, so we no longer need to memoize, because it will
    // never be thrown again.
    pingCache.delete(wakeable);
  }

  const eventTime = requestEventTime();
  markRootPinged(root, pingedLanes, eventTime);

  warnIfSuspenseResolutionNotWrappedWithActDEV(root);

  if (
    workInProgressRoot === root &&
    isSubsetOfLanes(workInProgressRootRenderLanes, pingedLanes)
  ) {
    // Received a ping at the same priority level at which we're currently
    // rendering. We might want to restart this render. This should mirror
    // the logic of whether or not a root suspends once it completes.

    // TODO: If we're rendering sync either due to Sync, Batched or expired,
    // we should probably never restart.

    // If we're suspended with delay, or if it's a retry, we'll always suspend
    // so we can always restart.
    if (
      workInProgressRootExitStatus === RootSuspendedWithDelay ||
      (workInProgressRootExitStatus === RootSuspended &&
        includesOnlyRetries(workInProgressRootRenderLanes) &&
        now() - globalMostRecentFallbackTime < FALLBACK_THROTTLE_MS)
    ) {
      // Restart from the root.
      prepareFreshStack(root, NoLanes);
    } else {
      // Even though we can't restart right now, we might get an
      // opportunity later. So we mark this render as having a ping.
      workInProgressRootPingedLanes = mergeLanes(
        workInProgressRootPingedLanes,
        pingedLanes
      );
    }
  }

  ensureRootIsScheduled(root, eventTime);
}
```

So the rationale is like this

1. when something is suspended, we try to render the fallback
2. but the promise might resolve before we are able to commit the fallback

what we do ? Retry is retry lane which is low priority, so we need some higher priority to ping the lanes and then restart the whole root?

# what is exactly ping?

We can see that `pingedLanes` is mostly used in `getNextLanes`.

```js
export function getNextLanes(root: FiberRoot, wipLanes: Lanes): Lanes {
  // Early bailout if there's no pending work left.
  const pendingLanes = root.pendingLanes;
  if (pendingLanes === NoLanes) {
    return NoLanes;
  }

  let nextLanes = NoLanes;

  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;

  // Do not work on any idle work until all the non-idle work has finished,
  // even if the work is suspended.
  const nonIdlePendingLanes = pendingLanes & NonIdleLanes;
  if (nonIdlePendingLanes !== NoLanes) {
    const nonIdleUnblockedLanes = nonIdlePendingLanes & ~suspendedLanes;
    if (nonIdleUnblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(nonIdleUnblockedLanes);
    } else {
      const nonIdlePingedLanes = nonIdlePendingLanes & pingedLanes;
      if (nonIdlePingedLanes !== NoLanes) {
        nextLanes = getHighestPriorityLanes(nonIdlePingedLanes);
      }
    }
  } else {
    // The only remaining work is Idle.
    const unblockedLanes = pendingLanes & ~suspendedLanes;
    if (unblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(unblockedLanes);
    } else {
      if (pingedLanes !== NoLanes) {
        nextLanes = getHighestPriorityLanes(pingedLanes);
      }
    }
  }
  ...
}
```

The logic here is

1. if there is non-idle pending lanes which are not suspended lanes, meaning it is higher priority, process them
2. if there is only suspendedLanes then process the pinged lanes first.

Kind of understand what is going on here now, it is like `ping some lane from suspended lane`.

# Let's run some demo

this PR is gold https://github.com/facebook/react/pull/15769

Refer to the test case in the PR.

# what is RootSuspendedWithDelay?

# The older version

https://github.com/facebook/react/blob/4e59d4f5d26a620a2c6e8804a589b227481a80aa/packages/react-reconciler/src/ReactFiberScheduler.old.js#L1710-L1740

```js
function pingSuspendedRoot(
  root: FiberRoot,
  thenable: Thenable,
  pingTime: ExpirationTime
) {
  // A promise that previously suspended React from committing has resolved.
  // If React is still suspended, try again at the previous level (pingTime).

  const pingCache = root.pingCache;
  if (pingCache !== null) {
    // The thenable resolved, so we no longer need to memoize, because it will
    // never be thrown again.
    pingCache.delete(thenable);
  }

  if (nextRoot !== null && nextRenderExpirationTime === pingTime) {
    // Received a ping at the same priority level at which we're currently
    // rendering. Restart from the root.
    nextRoot = null;
  } else {
    // Confirm that the root is still suspended at this level. Otherwise exit.
    if (isPriorityLevelSuspended(root, pingTime)) {
      // Ping at the original level
      markPingedPriorityLevel(root, pingTime);
      const rootExpirationTime = root.expirationTime;
      if (rootExpirationTime !== NoWork) {
        requestWork(root, rootExpirationTime);
      }
    }
  }
}
```

root pr https://github.com/facebook/react/pull/14429

```js
function pingSuspendedRoot(root: FiberRoot, pingTime: ExpirationTime) {
  // A promise that previously suspended React from committing has resolved.
  // If React is still suspended, try again at the previous level (pingTime).
  if (nextRoot !== null && nextRenderExpirationTime === pingTime) {
    // Received a ping at the same priority level at which we're currently
    // rendering. Restart from the root.
    nextRoot = null;
  } else {
    // Confirm that the root is still suspended at this level. Otherwise exit.
    if (isPriorityLevelSuspended(root, pingTime)) {
      // Ping at the original level
      markPingedPriorityLevel(root, pingTime);
      const rootExpirationTime = root.expirationTime;
      if (rootExpirationTime !== NoWork) {
        requestWork(root, rootExpirationTime);
      }
    }
  }
}
```

# need to reproduce

1. when promise is resolve and pinged some Lane
2. React happens to be rendering this Lane and about to render the fallback => could be triggered by something else

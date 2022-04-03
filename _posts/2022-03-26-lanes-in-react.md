---
layout: post
title: "What is Lanes in React source code? - React Source Code Walkthrough 21"
date: 2022-03-26 18:21:10 +0900
categories: React
image: /static/21-lanes.png
---

> you can watch my [youtube video for this topic](https://www.youtube.com/watch?v=yy8HRyhA_oQ)

- [Three priority systems](#three-priority-systems)
- [What is “Lane” ?](#what-is-lane-)
  - [Bitwise operation](#bitwise-operation)
  - [remember childLanes ?](#remember-childlanes-)
- [Look at performConcurrentWorkOnRoot() again](#look-at-performconcurrentworkonroot-again)
- [updateReducer()](#updatereducer)
- [Summary](#summary)
- [But What’s the point of Lanes? Let’s look at some example.](#but-whats-the-point-of-lanes-lets-look-at-some-example)
  - [Demo 1. Inputs blocked by rendering long list](#demo-1-inputs-blocked-by-rendering-long-list)
  - [Demo 2. Input not blocked by moving heavy work to transition lanes](#demo-2-input-not-blocked-by-moving-heavy-work-to-transition-lanes)
  - [Demo 3. use the internal API to schedule.](#demo-3-use-the-internal-api-to-schedule)

Previously we've seen how React schedules tasks by their priorities, a sample task would be `performConcurrentWorkOnRoot()`, which targets the fiber tree as whole, not the level of work on a single fiber.

The concurrent mode is cool because React is able to work on different work on each fiber by their priorities, this low level priority is implement with the concept of "Lane".

This might sounds jargon, don't worry, we'll dive into it and there are some examples at the end.

## Three priority systems

Take a look at `ensureRootIsScheduled()` again, this is one of the entries where scheduling method is called.

```js
// We use the highest priority lane to represent the priority of the callback.
const newCallbackPriority = getHighestPriorityLane(nextLanes);

if (newCallbackPriority === SyncLane) {
  ...
} else {
  let schedulerPriorityLevel;
  switch (lanesToEventPriority(nextLanes)) {
    case DiscreteEventPriority:
      schedulerPriorityLevel = ImmediateSchedulerPriority;
      break;
    case ContinuousEventPriority:
      schedulerPriorityLevel = UserBlockingSchedulerPriority;
      break;
    case DefaultEventPriority:
      schedulerPriorityLevel = NormalSchedulerPriority;
      break;
    case IdleEventPriority:
      schedulerPriorityLevel = IdleSchedulerPriority;
      break;
    default:
      schedulerPriorityLevel = NormalSchedulerPriority;
      break;
  }
  newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performConcurrentWorkOnRoot.bind(null, root),
  );
}

```

Interesting, from above code we can see that the scheduler priority is derived by

1. get the highest lane priority - `getHighestPriorityLane()`
2. if not SyncLane, map lanes to Event Priority, then map to Scheduler Priority.

Thus we have 3 priority systems

1. Scheduler priority - used to prioritize tasks in scheduler
2. Event priority - to mark priority of user event
3. Lane priority - to mark priority of work

Since they are for different purposes, they are separated but have logic of mapping like above, we'll cover event system in future episodes, so we don't touch much details here.

## What is "Lane" ?

In my youtube video about [setState](https://www.youtube.com/watch?v=svaUEHMuv9w), we've covered that a fiber holds a linked list of hooks, and for state hook, it has a update queue which would be run during update(rerender).

Here is the code where an update is created([source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.old.js#L2175-L2183))

```js
const lane = requestUpdateLane(fiber);

const update: Update<S, A> = {
  lane,
  action,
  hasEagerState: false,
  eagerState: null,
  next: (null: any),
};
```

Yep, notice there is field called `lane`?. **`Lane` is to mark the priority of the update, which we can also say mark the priority of a piece of work**.

Here are all the lanes in React, just some numbers which is easier to understand in binary forms.

```js
// Lane values below should be kept in sync with getLabelForLane(), used by react-devtools-timeline.
// If those values are changed that package should be rebuilt and redeployed.

export const TotalLanes = 31;

export const NoLanes: Lanes = /*                        */ 0b0000000000000000000000000000000;
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;

export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000001;

export const InputContinuousHydrationLane: Lane = /*    */ 0b0000000000000000000000000000010;
export const InputContinuousLane: Lanes = /*            */ 0b0000000000000000000000000000100;

export const DefaultHydrationLane: Lane = /*            */ 0b0000000000000000000000000001000;
export const DefaultLane: Lanes = /*                    */ 0b0000000000000000000000000010000;

const TransitionHydrationLane: Lane = /*                */ 0b0000000000000000000000000100000;
const TransitionLanes: Lanes = /*                       */ 0b0000000001111111111111111000000;
const TransitionLane1: Lane = /*                        */ 0b0000000000000000000000001000000;
const TransitionLane2: Lane = /*                        */ 0b0000000000000000000000010000000;
const TransitionLane3: Lane = /*                        */ 0b0000000000000000000000100000000;
const TransitionLane4: Lane = /*                        */ 0b0000000000000000000001000000000;
const TransitionLane5: Lane = /*                        */ 0b0000000000000000000010000000000;
const TransitionLane6: Lane = /*                        */ 0b0000000000000000000100000000000;
const TransitionLane7: Lane = /*                        */ 0b0000000000000000001000000000000;
const TransitionLane8: Lane = /*                        */ 0b0000000000000000010000000000000;
const TransitionLane9: Lane = /*                        */ 0b0000000000000000100000000000000;
const TransitionLane10: Lane = /*                       */ 0b0000000000000001000000000000000;
const TransitionLane11: Lane = /*                       */ 0b0000000000000010000000000000000;
const TransitionLane12: Lane = /*                       */ 0b0000000000000100000000000000000;
const TransitionLane13: Lane = /*                       */ 0b0000000000001000000000000000000;
const TransitionLane14: Lane = /*                       */ 0b0000000000010000000000000000000;
const TransitionLane15: Lane = /*                       */ 0b0000000000100000000000000000000;
const TransitionLane16: Lane = /*                       */ 0b0000000001000000000000000000000;

const RetryLanes: Lanes = /*                            */ 0b0000111110000000000000000000000;
const RetryLane1: Lane = /*                             */ 0b0000000010000000000000000000000;
const RetryLane2: Lane = /*                             */ 0b0000000100000000000000000000000;
const RetryLane3: Lane = /*                             */ 0b0000001000000000000000000000000;
const RetryLane4: Lane = /*                             */ 0b0000010000000000000000000000000;
const RetryLane5: Lane = /*                             */ 0b0000100000000000000000000000000;

export const SomeRetryLane: Lane = RetryLane1;

export const SelectiveHydrationLane: Lane = /*          */ 0b0001000000000000000000000000000;

const NonIdleLanes = /*                                 */ 0b0001111111111111111111111111111;

export const IdleHydrationLane: Lane = /*               */ 0b0010000000000000000000000000000;
export const IdleLane: Lanes = /*                       */ 0b0100000000000000000000000000000;

export const OffscreenLane: Lane = /*                   */ 0b1000000000000000000000000000000;
```

Just like lanes on the road, the rule is to drive on different lanes based on your speed, meaning the smaller the lane is the more urgent the work is, the higher priority it is. so `SyncLane` is `1` here.

There are many lanes here. We don't dive into each of them but rather understand how it works generally in this episode.

## Bitwise operation

The lanes are just numbers, there are a lot of bitwise operations in React source code, let's get familiar with that.

Some examples which explain themselves.

```js
export function includesSomeLane(a: Lanes | Lane, b: Lanes | Lane) {
  return (a & b) !== NoLanes;
}

export function isSubsetOfLanes(set: Lanes, subset: Lanes | Lane) {
  return (set & subset) === subset;
}

export function mergeLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a | b;
}

export function removeLanes(set: Lanes, subset: Lanes | Lane): Lanes {
  return set & ~subset;
}

export function intersectLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a & b;
}
```

### remember `childLanes` ?

In [How does React bailout work in reconciliation]({% post_url 2022-01-08-how-does-bailout-work %}), we've also touched a bit of `lanes` and `childLanes` of a fiber.

So, each fiber knows

1. priorities for the work of itself - `lanes`
2. priorities for the work of its descendant - `childLanes`.

## Look at performConcurrentWorkOnRoot() again

So, here is basically the flow of how a work is scheduled and run

1. get the nextLanes of the fiber tree
2. map it to scheduler priority
3. schedule the task to reconcile
4. reconciliation happens, process the work from root

I guess the magic lies in actually reconciliation, that is where lane info is used.

```js
let lanes = getNextLanes(
  root,
  root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
);

...
prepareFreshStack(root, lanes);
```

`prepareFreshStack()` means restart the recociliation, remember there is a cursor(`workInProgress`) to track the current fiber, usually React would pause and resume from previous position, but in case of error or some weird cases, we need to give up current jobs done and redo them from beginining, this is what `fresh` means.

```js
function prepareFreshStack(root: FiberRoot, lanes: Lanes) {
  root.finishedWork = null;
  root.finishedLanes = NoLanes;

  const timeoutHandle = root.timeoutHandle;
  if (timeoutHandle !== noTimeout) {
    // The root previous suspended and scheduled a timeout to commit a fallback
    // state. Now that we have additional work, cancel the timeout.
    root.timeoutHandle = noTimeout;
    // $FlowFixMe Complains noTimeout is not a TimeoutID, despite the check above
    cancelTimeout(timeoutHandle);
  }

  if (workInProgress !== null) {
    let interruptedWork = workInProgress.return;
    while (interruptedWork !== null) {
      unwindInterruptedWork(interruptedWork, workInProgressRootRenderLanes);
      interruptedWork = interruptedWork.return;
    }
  }
  workInProgressRoot = root;
  workInProgress = createWorkInProgress(root.current, null);
  workInProgressRootRenderLanes =
    subtreeRenderLanes =
    workInProgressRootIncludedLanes =
      lanes;
  workInProgressRootExitStatus = RootIncomplete;
  workInProgressRootFatalError = null;
  workInProgressRootSkippedLanes = NoLanes;
  workInProgressRootInterleavedUpdatedLanes = NoLanes;
  workInProgressRootRenderPhaseUpdatedLanes = NoLanes;
  workInProgressRootPingedLanes = NoLanes;

  enqueueInterleavedUpdates();
}
```

We can see in `prepareFreshStack()`, just some variables are reset. There are quite a few of them about lanes.

1. workInProgressRootRenderLanes
2. subtreeRenderLanes
3. workInProgressRootIncludedLanes
4. workInProgressRootSkippedLanes
5. workInProgressRootInterleavedUpdatedLanes
6. workInProgressRootRenderPhaseUpdatedLanes
7. workInProgressRootPingedLanes

Ok, no clues what they are for now, but `workInProgressRootRenderLanes` looks simple to dive into.

```js
// The lanes we're rendering
let workInProgressRootRenderLanes: Lanes = NoLanes;
```

As the comment says itself, this is the lanes we're rendering.

There are a few places it is used, for example here:

```js
export function requestUpdateLane(fiber: Fiber): Lane {
  // Special cases
  const mode = fiber.mode;
  if ((mode & ConcurrentMode) === NoMode) {
    return (SyncLane: Lane);
  } else if (
    !deferRenderPhaseUpdateToNextBatch &&
    (executionContext & RenderContext) !== NoContext &&
    workInProgressRootRenderLanes !== NoLanes
  ) {
    // This is a render phase update. These are not officially supported. The
    // old behavior is to give this the same "thread" (lanes) as
    // whatever is currently rendering. So if you call `setState` on a component
    // that happens later in the same render, it will flush. Ideally, we want to
    // remove the special case and treat them as if they came from an
    // interleaved event. Regardless, this pattern is not officially supported.
    // This behavior is only a fallback. The flag only exists until we can roll
    // out the setState warning, since existing code might accidentally rely on
    // the current behavior.
    return pickArbitraryLane(workInProgressRootRenderLanes);
  }

  // Updates originating inside certain React methods, like flushSync, have
  // their priority set by tracking it with a context variable.
  //
  // The opaque type returned by the host config is internally a lane, so we can
  // use that directly.
  // TODO: Move this type conversion to the event priority module.
  const updateLane: Lane = (getCurrentUpdatePriority(): any);
  if (updateLane !== NoLane) {
    return updateLane;
  }

  // This update originated outside React. Ask the host environment for an
  // appropriate priority, based on the type of event.
  //
  // The opaque type returned by the host config is internally a lane, so we can
  // use that directly.
  // TODO: Move this type conversion to the event priority module.
  const eventLane: Lane = (getCurrentEventPriority(): any);
  return eventLane;
}
``;
```

Aha, notice it is `requestUpdateLane()`? which is mentioned at the beginning. It is a bit hard to understand what is going on in above function, but it is clear that current rendering lanes somewhat affect the lanes scheduled while rendering.

OK, let's go back to `performConcurrentWorkOnRoot()`.

```js
// Determine the next lanes to work on, using the fields stored
// on the root.
let lanes = getNextLanes(
  root,
  root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
);
```

This decides what lanes would be worked on, `getNextLanes()` is pretty complex, we'll skip here, just keep in mind that for the very basic cases `getNextLanes()` picks up the the lane of highest priority.

```js
if (lanes === NoLanes) {
  // Defensive coding. This is never expected to happen.
  return null;
}

// We disable time-slicing in some cases: if the work has been CPU-bound
// for too long ("expired" work, to prevent starvation), or we're in
// sync-updates-by-default mode.
// TODO: We only check `didTimeout` defensively, to account for a Scheduler
// bug we're still investigating. Once the bug in Scheduler is fixed,
// we can remove this, since we track expiration ourselves.
const shouldTimeSlice =
  !includesBlockingLane(root, lanes) &&
  !includesExpiredLane(root, lanes) &&
  (disableSchedulerTimeoutInWorkLoop || !didTimeout);
let exitStatus = shouldTimeSlice
  ? renderRootConcurrent(root, lanes)
  : renderRootSync(root, lanes);
```

Interesting, we can see that even in concurrent mode, it might fall back to sync mode in some cases, for example, if the lanes includes blocking lanes or some lanes are expired.

Let's skip the details of the detailed function and move on.

## updateReducer()

As we have covered before, `useState()` is mapped to `mountState()` for initial render and `updateState()` in the updates.

The state updates happens in `updateState()`.

```js
function updateState<S>(
  initialState: (() => S) | S
): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, (initialState: any));
}
```

Internally `updateReducer()` is used. ([source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.old.js#L754))

```js
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: (I) => S
): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;

  if (queue === null) {
    throw new Error(
      "Should have a queue. This is likely a bug in React. Please file an issue."
    );
  }

  queue.lastRenderedReducer = reducer;

  const current: Hook = (currentHook: any);

  // The last rebase update that is NOT part of the base state.
  let baseQueue = current.baseQueue;

  // The last pending update that hasn't been processed yet.
  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
    // We have new updates that haven't been processed yet.
    // We'll add them to the base queue.
    if (baseQueue !== null) {
      // Merge the pending queue and the base queue.
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;
    }
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }

  if (baseQueue !== null) {
    // We have a queue to process.
    const first = baseQueue.next;
    let newState = current.baseState;

    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast = null;
    let update = first;
    do {
      const updateLane = update.lane;
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // Priority is insufficient. Skip this update. If this is the first
        // skipped update, the previous update/state is the new base
        // update/state.
        const clone: Update<S, A> = {
          lane: updateLane,
          action: update.action,
          hasEagerState: update.hasEagerState,
          eagerState: update.eagerState,
          next: (null: any),
        };
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        // Update the remaining priority in the queue.
        // TODO: Don't need to accumulate this. Instead, we can remove
        // renderLanes from the original lanes.
        currentlyRenderingFiber.lanes = mergeLanes(
          currentlyRenderingFiber.lanes,
          updateLane
        );
        markSkippedUpdateLanes(updateLane);
      } else {
        // This update does have sufficient priority.

        if (newBaseQueueLast !== null) {
          const clone: Update<S, A> = {
            // This update is going to be committed so we never want uncommit
            // it. Using NoLane works because 0 is a subset of all bitmasks, so
            // this will never be skipped by the check above.
            lane: NoLane,
            action: update.action,
            hasEagerState: update.hasEagerState,
            eagerState: update.eagerState,
            next: (null: any),
          };
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }

        // Process this update.
        if (update.hasEagerState) {
          // If this update is a state update (not a reducer) and was processed eagerly,
          // we can use the eagerly computed state
          newState = ((update.eagerState: any): S);
        } else {
          const action = update.action;
          newState = reducer(newState, action);
        }
      }
      update = update.next;
    } while (update !== null && update !== first);

    if (newBaseQueueLast === null) {
      newBaseState = newState;
    } else {
      newBaseQueueLast.next = (newBaseQueueFirst: any);
    }

    // Mark that the fiber performed work, but only if the new state is
    // different from the current state.
    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();
    }

    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;

    queue.lastRenderedState = newState;
  }

  return [hook.memoizedState, dispatch];
}
```

A big one, but let's just focus on the core part

```js
do {
  const updateLane = update.lane;
  if (!isSubsetOfLanes(renderLanes, updateLane)) {
    // Priority is insufficient. Skip this update. If this is the first
    // skipped update, the previous update/state is the new base
    // update/state.
    ...
  }
  update = update.next;
} while (update !== null && update !== first);
```

Yeah, it loop through the updates and checks the lanes by `isSubsetOfLanes`, `renderLanes` is set in `renderWithHooks()`. [source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.old.js#L376), tracing back, the root function call is in `performUnitOfWork()`.

```
next = beginWork(current, unitOfWork, subtreeRenderLanes);
```

Phew, end of talk. This is a lot so far, we've seen how the lanes work generally.

## Summary

1. when events happens on fibers, updates are created with Lane info, which is determined by a few factors.
2. ancester fibers are marked with childLanes, so for any fiber, we can get the lane info for desendant nodes.
3. get the highest priority lanes from root -> map it to scheduler priority -> schedule a task in scheduler to reconcile the fiber tree
4. in reconciliation, pick the highest priority lanes to work on - current rendering lanes
5. traverse throught the fiber tree, check the updates on hooks, run the updates that have lanes included in the rendering lanes.

Thus we are able to run separately for multiple updates on single fiber.

## But What's the point of Lanes? Let's look at some example.

A demo explains more than a thousand words.

### Demo 1. Inputs blocked by rendering long list

Open [the first demo](/demos/react/lanes-priority/without-transition.html), type something in the input, you can feel the lagging, input field is not responding.

![](/static/lanes-without-transition.gif)

It is laggy because we enfored the delay for the rendering of each cell.

```jsx
function Cell() {
  const start = Date.now();
  while (Date.now() - start < 1) {}
  return <span className={`cell ${COLORS[Math.round(Math.random())]}`} />;
}

function _Cells() {
  return (
    <div className="cells">
      {new Array(1000).fill(0).map((_, index) => (
        <Cell key={index} />
      ))}
    </div>
  );
}
const Cells = React.memo(_Cells);
function App() {
  const [text, setText] = useState("");
  return (
    <div className="app">
      <input
        type="text"
        value={text}
        onChange={(e) => {
          setText(e.target.value);
        }}
      />
      <Cells text={text} />
    </div>
  );
}
```

We can improve the case by separating the lanes for updating `<Cells>`, by using `startTransion()`, see the second demo.

### Demo 2. Input not blocked by moving heavy work to transition lanes

Open [the second demo](/demos/react/lanes-priority/with-transition.html) to give it a try

![](/static/lanes-with-transition.gif)

We can see that the input is instantly responding, while the cells are rendered later.

This is because we moved the update of `<Cells/>` to transition lanes.

```jsx
function App() {
  const [text, setText] = useState("");
  const deferrredText = React.useDeferredValue(text);
  return (
    <div className="app">
      <input
        type="text"
        value={text}
        onChange={(e) => {
          setText(e.target.value);
        }}
      />
      <Cells text={deferrredText} />
    </div>
}
```

The trick here is `useDeferredValue()`, which put updates in transition lanes. For details of this built-in method, watch my video [How does React.useDeferredValue() works?](https://www.youtube.com/watch?v=db31-3xw_3U)

Also open the devtool, you can see the difference of these two.

For the first one:

```js
pendingLanes 0000000000000000000000000000001
pendingLanes 0000000000000000000000000000001
performSyncWorkOnRoot()
lanes to work on  0000000000000000000000000000001
workLoopSync
pendingLanes 0000000000000000000000000000000
pendingLanes 0000000000000000000000000000000
```

You can see there is only one lane SyncLane, so the update for the input and the cells are processed in the same batch.

While for the second demo is a bit different.

```js
pendingLanes 0000000000000000000000000000001
pendingLanes 0000000000000000000000000000001
performSyncWorkOnRoot()
lanes to work on  0000000000000000000000000000001
workLoopSync
pendingLanes 0000000000000000000000001000000
pendingLanes 0000000000000000000000001000000
pendingLanes 0000000000000000000000001000000
performConcurrentWorkOnRoot()
pendingLanes 0000000000000000000000001000000
```

We can see there is two round, first is SyncLane for the input, but for the Cells, it is `TransitionLane1`.

### Demo 3. use the internal API to schedule.

My [3rd demo](/demos/react/lanes-priority/with-schedule-api.html) is also easy to understand.

```jsx
function App() {
  const [num, setNum] = React.useState(1);
  const renders = React.useRef([]);
  renders.current.push(num);
  return (
    <div>
      <button
        onClick={() => {
          setCurrentUpdatePriority(4);
          setNum((num) => num + 1);

          setCurrentUpdatePriority(1);
          setNum((num) => num * 10);
        }}
      >
        click me
      </button>
      {renders.current.map((log, i) => (
        <p key={i}>{log}</p>
      ))}
    </div>
  );
}
```

We called `seState()` twice at the same time, but each with different update priority (lane), first call is `InputContinuousLane`, the second one is `SyncLane`.

So what do you expect for the result ?

If we don't consider priority, we might think they are process together so `1 -> 20`.

Actual result is `1 -> 10 -> 20`.

Open devtool and click the button, we would see what is going on

```bash
pendingLanes 0000000000000000000000000000100
pendingLanes 0000000000000000000000000000101
pendingLanes 0000000000000000000000000000101
performSyncWorkOnRoot()
lanes to work on  0000000000000000000000000000001
workLoopSync
render App() with state  10
pendingLanes 0000000000000000000000000000100
pendingLanes 0000000000000000000000000000100
performConcurrentWorkOnRoot()
pendingLanes 0000000000000000000000000000100
lanes to work on 0000000000000000000000000000100
shouldTimeSlice false
workLoopSync
render App() with state  20
pendingLanes 0000000000000000000000000000000
pendingLanes 0000000000000000000000000000000
```

First we processed SyncLane, so `1 * 10 = 10`, then process the rest lane, notice SyncLane hook update still needs to be run for the consistency, so `(1 + 1) * 10 = 20`.

That's it for this episode, hope it helps you understand better of React internals.

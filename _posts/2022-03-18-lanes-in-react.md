---
layout: post
title: "What is Lanes in React source code? - React Source Code Walkthrough 21"
date: 2022-03-18 18:21:10 +0900
categories: React
image: /static/react-scheduler.png
---

Previously we've seen how React schedules tasks by their priorities.

How it is translated into the small "work" in reconciler?

## `ensureRootIsScheduled()` again

This is one of the entries where scheduling method is called, there is some logic as below

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

Interesting. So `Lanes` are different for schedulerPriority. In previous post we've already known that there are 5 scheduler priorities, looking at above switch , we can see there are more `Lane` priority than scheduler priority.

## What is "Lanes" ?

```js
const nextLanes = getNextLanes(
  root,
  root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
);
```

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

Just like lanes on the road, the rule is to drive on different lanes based on your speed, meaning it is more urgent

The smaller the lane is, the higher priority it is . so `SyncLane` is `1` here.

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

## Lanes for each fiber node

For the fiber tree, each node has its lanes, and it is accumulated along the path.

meaning each fiber has all the lane information about its descendent.

We've touched part of this logic in the bailout [TODO]

# So the flow is like this

1. get the nextLanes of the fiber tree
2. map it to scheduler priority
3. schedule the task to reconcile in scheduler
4. reconciliation happens, might be interrupted, might resume

So the magic lies in actually reconciliation, how the lanes are handled.

# performConcurrentWorkOnRoot() again

Let's see how lanes are used in performConcurrentWorkOnRoot().

```js
let lanes = getNextLanes(
  root,
  root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
);
```

It passes down the lanes to `renderRootConcurrent()`, there is one place lanes is sued.

```js
prepareFreshStack(root, lanes);
```

in prepareFreshStack()

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

It looks like a reset for some global variables, ther are 4 of them about lanes

1. workInProgressRootRenderLanes
2. subtreeRenderLanes
3. workInProgressRootIncludedLanes
4. workInProgressRootSkippedLanes
5. workInProgressRootInterleavedUpdatedLanes
6. workInProgressRootRenderPhaseUpdatedLanes
7. workInProgressRootPingedLanes

Ok, I have totally no clues what these are for. Let's keep in mind that there are a bunch of tracking of lanes.

To proceed, we first pick `workInProgressRootRenderLanes` to see where they are used.

# workInProgressRootRenderLanes

```js
// The lanes we're rendering
let workInProgressRootRenderLanes: Lanes = NoLanes;
```

As the comment says itself, this is the lanes we're rendering. Lanes and Lane are both just numebers.

There are a few places it is used, let's focus on the most important one

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

# take one lane

To understand how the lanes work, it is easier to focus on one of the lanes.

# InputContinuousLane

Lane means the priority for Work on fiber tree, then there must be some places the lanes are set.

Starting from the events.

```js
export const ContinuousEventPriority: EventPriority = InputContinuousLane;
```

InputContinuousLane is mapped to `ContinuousEventPriority`.

```js
export function createEventListenerWrapperWithPriority(
  targetContainer: EventTarget,
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags
): Function {
  const eventPriority = getEventPriority(domEventName);
  let listenerWrapper;
  switch (eventPriority) {
    case DiscreteEventPriority:
      listenerWrapper = dispatchDiscreteEvent;
      break;
    case ContinuousEventPriority:
      listenerWrapper = dispatchContinuousEvent;
      break;
    case DefaultEventPriority:
    default:
      listenerWrapper = dispatchEvent;
      break;
  }
  return listenerWrapper.bind(
    null,
    domEventName,
    eventSystemFlags,
    targetContainer
  );
}
```

React has defined different event priority by event name. For mouse events, mostly they are set with
`ContinuousEventPriority`, for `message` event it is a bit exception, it uses current scheduler priority.

```js
export function getEventPriority(domEventName: DOMEventName): * {
  switch (domEventName) {
    // Used by SimpleEventPlugin:
    case "cancel":
    case "click":
    case "close":
    case "contextmenu":
    case "copy":
    case "cut":
    case "auxclick":
    case "dblclick":
    case "dragend":
    case "dragstart":
    case "drop":
    case "focusin":
    case "focusout":
    case "input":
    case "invalid":
    case "keydown":
    case "keypress":
    case "keyup":
    case "mousedown":
    case "mouseup":
    case "paste":
    case "pause":
    case "play":
    case "pointercancel":
    case "pointerdown":
    case "pointerup":
    case "ratechange":
    case "reset":
    case "resize":
    case "seeked":
    case "submit":
    case "touchcancel":
    case "touchend":
    case "touchstart":
    case "volumechange":
    // Used by polyfills:
    // eslint-disable-next-line no-fallthrough
    case "change":
    case "selectionchange":
    case "textInput":
    case "compositionstart":
    case "compositionend":
    case "compositionupdate":
    // Only enableCreateEventHandleAPI:
    // eslint-disable-next-line no-fallthrough
    case "beforeblur":
    case "afterblur":
    // Not used by React but could be by user code:
    // eslint-disable-next-line no-fallthrough
    case "beforeinput":
    case "blur":
    case "fullscreenchange":
    case "focus":
    case "hashchange":
    case "popstate":
    case "select":
    case "selectstart":
      return DiscreteEventPriority;
    case "drag":
    case "dragenter":
    case "dragexit":
    case "dragleave":
    case "dragover":
    case "mousemove":
    case "mouseout":
    case "mouseover":
    case "pointermove":
    case "pointerout":
    case "pointerover":
    case "scroll":
    case "toggle":
    case "touchmove":
    case "wheel":
    // Not used by React but could be by user code:
    // eslint-disable-next-line no-fallthrough
    case "mouseenter":
    case "mouseleave":
    case "pointerenter":
    case "pointerleave":
      return ContinuousEventPriority;
    case "message": {
      // We might be in the Scheduler callback.
      // Eventually this mechanism will be replaced by a check
      // of the current priority on the native scheduler.
      const schedulerPriority = getCurrentSchedulerPriorityLevel();
      switch (schedulerPriority) {
        case ImmediateSchedulerPriority:
          return DiscreteEventPriority;
        case UserBlockingSchedulerPriority:
          return ContinuousEventPriority;
        case NormalSchedulerPriority:
        case LowSchedulerPriority:
          // TODO: Handle LowSchedulerPriority, somehow. Maybe the same lane as hydration.
          return DefaultEventPriority;
        case IdleSchedulerPriority:
          return IdleEventPriority;
        default:
          return DefaultEventPriority;
      }
    }
    default:
      return DefaultEventPriority;
  }
}
```

So what is in `dispatchContinuousEvent()`

```js
function dispatchContinuousEvent(
  domEventName,
  eventSystemFlags,
  container,
  nativeEvent
) {
  const previousPriority = getCurrentUpdatePriority();
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = 0;
  try {
    setCurrentUpdatePriority(ContinuousEventPriority);
    dispatchEvent(domEventName, eventSystemFlags, container, nativeEvent);
  } finally {
    setCurrentUpdatePriority(previousPriority);
    ReactCurrentBatchConfig.transition = prevTransition;
  }
}
```

So when a continuous event is fired, it update `currentUpdatePriority`, and set it back when dispatching is done.

For details of synthetic event system, we'll do it some other day.

# So let's try some example.

if we are use `useState()` in a continuous event listener, what would happen?

Let's dive into `useState()`. We've covered the useState in Video[TODO]

if bind on `onClick`

1. the event is dispatched as `DiscreteEventPriority`, which is `SyncLane`, notice `SyncLane` doesn't mean synchornous, it is just meaning it is high proirity, this is reasonable, when someone clicks, we must give the quickest response.

2. during the event firing. `setState()` is called.

```js
var lane = requestUpdateLane(fiber);
var update = {
  lane: lane,
  action: action,
  hasEagerState: false,
  eagerState: null,
  next: null,
};
var root = scheduleUpdateOnFiber(fiber, lane, eventTime);
```

3. in `requestUpdateLane()`, `getCurrentUpdatePriority()` will return SyncLane.
4. an update with `SyncLane` will be scheduled through `scheduleUpdateOnFiber`

in scheduleUpdateOnFiber()

```js
markUpdateLaneFromFiberToRoot(fiber, lane);
ensureRootIsScheduled(root, eventTime);
```

Looks familiar to previous.

well if it is continuous event, lane would be `100`.

So if there are different kinds of lane waiting, then `SyncLane` will be first run.

# How can we create such exmaple?

It is hard for me to consider an actual example except the transition API.

Since this needs to schedule a bunch of events of different levels together.

Let's try manually create some cases.

```js
function App() {
  const [state, setState] = React.useState(0);
  console.log("updateReducer, got", state);
  return (
    <div
      onClick={() => {
        setCurrentUpdatePriority(2);
        setState("priority 2");
        setCurrentUpdatePriority(1);
        setState("priority 3");
        // setCurrentUpdatePriority(1)
        setTimeout(() => {
          setState("priority 1");
        }, 0);
      }}
    >
      {state}
      <A />
    </div>
  );
}
```

---
layout: post
title: "How does act() work in React? | React Source Code Walkthrough 24"
date: 2022-05-15 18:21:10 +0900
categories: React
image: /static/act.png
---

React has a few [built-in test utils](https://reactjs.org/docs/test-utils.html), usually we don't use them directly but through some well integrated testing libraries and frameworks.

Since we have already covered a few topics of React internals, it would be interesting to see how these built-in test utils work internally.

We are going to look at `act()` today.

- [1. demo of act()](#1-demo-of-act)
- [2. how does act() work?](#2-how-does-act-work)
- [3. how callback is scheduled in the act queue ?](#3-how-callback-is-scheduled-in-the-act-queue-)
- [4. How act queue is exported ?](#4-how-act-queue-is-exported-)
- [5. Summary](#5-summary)

## 1. demo of act()

According to [official document](https://reactjs.org/docs/testing-recipes.html#act),

> When writing UI tests, tasks like rendering, user events, or data fetching can be considered as “units” of interaction with a user interface. react-dom/test-utils provides a helper called act() that makes sure all updates related to these “units” have been processed and applied to the DOM before you make any assertions

Simply put things scheduled in `act()` should be run synchously.

Here is [a demo](/demos/react/act/without-act.html).

```jsx
function App() {
  useEffect(() => {
    console.log("effect");
  });

  return null;
}

const root = ReactDOM.createRoot(document.getElementById("container"));
root.render(<App />);
console.log("after render");
```

Since the rendering of concurrent mode is async and so are passive effects, we could expect the order of logs to be

```
after render
effect
```

You can [try it out here](/demos/react/act/without-act.html).

Now let's wrap the rendering into act, [try it out](/demos/react/act/with-act.html)

```jsx
function App() {
  useEffect(() => {
    console.log("effect");
  });

  return null;
}

const root = ReactDOM.createRoot(document.getElementById("container"));
React.unstable_act(() => root.render(<App />));
console.log("after render");
```

Now the order changes, effects are flushed synchrously.

```
effect
after render
```

This would be useful during test, since the asynchrous behavior is inside of the blackbox of React,
it could be easier to assert without await stuff.

## 2. how does act() work?

We've briefly explained how effect is run in [The lifecycle of effect hooks in React]({% post_url 2022-01-17-lifecycle-of-effect-hook %}).

Simply put, React runtime creates a new version of fiber tree and commits necessary changes to DOM, if there are passive effects, **Scheduler schedules a task to run those effects**. Here is the [code in React repo](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L2132).

The concept of Scheduler is that it keeps working on the tasks but without blocking the main thread for too long, the tasks are scheduled asynchrously through `setImmediate()` or `MessageChannel` callback or `setTimeout`. I explained it in this post - [How Scheduler works]({% post_url 2022-03-16-how-react-scheduler-works %}).

So in order to make `act()` work, we need to change the behavior of Scheduler to sync under some condition. Well React does something like this but rather than changing the internals of Scheduler, it bypasses Scheduler.

[ReactAct.js](https://github.com/facebook/react/blob/main/packages/react/src/ReactAct.js) is the module of `act()`, it is a bit long, let's try to break it down.

1. `ReactCurrentActQueue` is very important, we can think of it as something similar to the task queue in Scheduler.
2. when `act(callback)` is called, the callback is called [here](https://github.com/facebook/react/blob/main/packages/react/src/ReactAct.js#L38)
3. `performConcurrentWorkOnRoot()` is scheduled in the act quque.
4. the queue is flushed by `flushActQueue()`, [here](https://github.com/facebook/react/blob/main/packages/react/src/ReactAct.js#L121)
   - `performConcurrentWorkOnRoot()` results in callback of `flushPassiveEffects()` being pushed into the Act Queue.
   - `flushActQueue()` continues to flush all of the tasks.

The above flow seems straightforward, except we have one big question:

## 3. how callback is scheduled in the act queue ?

First `ReactCurrentActQueue` is defined and exported from [ReactCurrentActQueue.js](https://github.com/facebook/react/blob/main/packages/react/src/ReactCurrentActQueue.js).

```js
type RendererTask = (boolean) => RendererTask | null;

const ReactCurrentActQueue = {
  current: (null: null | Array<RendererTask>),

  // Used to reproduce behavior of `batchedUpdates` in legacy mode.
  isBatchingLegacy: false,
  didScheduleLegacyUpdate: false,
};

export default ReactCurrentActQueue;
```

Simply it holds reference of `current` to an array.

And we can find the call stack of how `performConcurrentWorkOnRoot()` is scheduled easily by breakpoint.

![](/static/act-2.png)

All the magic lies in `scheduleCallback()`, code [here](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L3137)

```js
function scheduleCallback(priorityLevel, callback) {
  if (__DEV__) {
    // If we're currently inside an `act` scope, bypass Scheduler and push to
    // the `act` queue instead.
    const actQueue = ReactCurrentActQueue.current;
    if (actQueue !== null) {
      actQueue.push(callback);
      return fakeActCallbackNode;
    } else {
      return Scheduler_scheduleCallback(priorityLevel, callback);
    }
  } else {
    // In production, always call Scheduler. This function will be stripped out.
    return Scheduler_scheduleCallback(priorityLevel, callback);
  }
}
```

We can clearly see that

1. if the act queue is there, the the callback is pushed in the queue
2. otherwise it goes into Scheduler.

After `performConcurrentWorkOnRoot()`, in the commit phase, we find there are passive effects to be flushed, [code here](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L2137-L2159).

```js
if (
  (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
  (finishedWork.flags & PassiveMask) !== NoFlags
) {
  if (!rootDoesHavePassiveEffects) {
    rootDoesHavePassiveEffects = true;
    pendingPassiveEffectsRemainingLanes = remainingLanes;
    // workInProgressTransitions might be overwritten, so we want
    // to store it in pendingPassiveTransitions until they get processed
    // We need to pass this through as an argument to commitRoot
    // because workInProgressTransitions might have changed between
    // the previous render and commit if we throttle the commit
    // with setTimeout
    pendingPassiveTransitions = transitions;
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

We can see `scheduleCallback()` is called again, meaning there is a new callback being pushed into the act queue, so the queue could be growing while being processed.

## 4. How act queue is exported ?

This is interesting, for the React runtime as explained above, the queue is imported under [ReactSharedInternals](https://github.com/facebook/react/blob/4c03bb6ed01a448185d9a1554229208a9480560d/packages/shared/ReactSharedInternals.js).

```js
import * as React from "react";

const ReactSharedInternals =
  React.__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED;

export default ReactSharedInternals;
```

And ... yeah, it is through the global `__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED`, ok, good naming.

```js
export {
  ...
  ReactSharedInternals as __SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED,
  ...
};
```

It is exported as an alias to [ReactSharedInternals](https://github.com/facebook/react/blob/4c03bb6ed01a448185d9a1554229208a9480560d/packages/react/src/ReactSharedInternals.js).

```js
if (__DEV__) {
  ReactSharedInternals.ReactDebugCurrentFrame = ReactDebugCurrentFrame;
  ReactSharedInternals.ReactCurrentActQueue = ReactCurrentActQueue;
}
```

So the code in the runtime is guarded by `__DEV__`, which is going to be stripped in the production build.

Why not just importing it? I don't know...yet, it is more of a coding convention thing I think, tell me if you know more about it.

## 5. Summary

Cool, that's it for `act()`, let's summarize a bit.

1. when `act()` is called, a shared callback queue - `ReactCurrentActQueue` is initialized.
2. in React runtime, if under `__DEV__` and `ReactCurrentActQueue` is not empty, the callbacks scheduled are pushed into the queue rather than scheduled in the Scheduler.
3. in `act()`, `ReactCurrentActQueue` is processed and flushed until it becomes empty.

This answers why effects are flushed synchrously if under `act()`, since the tasks scheduled to `flushPassiveEffects()` are going to the queue, React Scheduler is bypassed.

---
layout: post
title: "How React Scheduler works? - React Source Code Walkthrough 20"
date: 2022-03-26 18:21:10 +0900
categories: React
image: /static/21-lanes.png
---

- [1. To Begin With](#1-to-begin-with)
- [2. Background knowledge](#2-lets-first-start-with-some-background-knowledge)
  - [2.1 Event Loop](#21-event-loop)
  - [2.2 setimmediate()](#22-setimmediate-to-schedule-a-new-task-without-blocking-the-rendering)
  - [2.3 Priority Queue](#23-priority-queue)
- [3. Call stack of workLoopConcurrent](#3-call-stack-of-workloopconcurrent)
  - [3.1 return value of performConcurrentWorkOnRoot()](#31-performconcurrentworkonroot-returns-a-closure-of-itself-if-interrupted)
- [4. Scheduler](#4-scheduler)
  - [4.1 scheduleCallback() - Scheduler schedules tasks by exipriationTime](#41-schedulecallback---scheduler-schedules-tasks-by-exipriationtime)
  - [4.2 requestHostCallback()](#42-requesthostcallback)
  - [4.3 flushWork()](#43-flushwork)
  - [4.4 workLoop() - the core of Scheduler](#44-workloop---the-core-of-scheduler)
  - [4.5 how shouldYield() work?](#45-how-shouldyield-work)
- [5. Summary](#5-summary)

## 1. To Begin With

Let's start with following piece of code([source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L1743)), we've already covered this part in [our very first episode of this series](https://www.youtube.com/watch?v=OcB3rTln-fI&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=1).

```js
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

In a word, React internally works on each fiber on the fiber tree, `workInProgress` is to track current position, the traversal algorithm is already explained [in my previous post]({% post_url 2022-01-16-fiber-traversal-in-react %}).

`workLoopSync()` is very easy to understand, since it is `sync`, there is no interrupting of our work, so React just keep working in a while loop.

Things are different in concurrent mode.([source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L1831))

```js
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

In concurrent mode, tasks with higher priorities could interrupt tasks with lower priority, we need a way to interrupt the tasks and resume, that is what `shouldYield()` does the trick, but obviously there is more than that.

## 2. Let's first start with some background knowledge

### 2.1 Event Loop

To be honest, I cannot explain this very well, I suggest you read [the explanation from javascript.info](https://javascript.info/event-loop) or watch [this nice video from Jake Archibald](https://twitter.com/jaffathecake/status/961980260194684928).

Simply speaking, JavaScript engine would do something as below

1. grab the task(macrotask) from the task queue and run it
2. if there are microtasks scheduled, run them
3. check if needs rerendering and render
4. repeat 1 if there are more task or wait for more task.

The `loop` is very self explanatory, because there is actually sort of loop there.

### 2.2 setImmediate() to schedule a new task without blocking the rendering

To schedule some task without blocking rendering(the step 3 above), we're already familiar with the trick of `setTimeout(callback, 0)`, it schedules a new macrotask.

But there is event better API [setImmediate()](https://developer.mozilla.org/en-US/docs/Web/API/Window/setImmediate) but it is only available in IE and node.js.

It is better because `setTimeout()` actually hav [minimum of about 4ms in nested calls](https://javascript.info/settimeout-setinterval), `setImmediate()` doesn't have the delay.

OK, we are ready to touch the first piece of code in React Scheduler [source](https://github.com/facebook/react/blob/main/packages/scheduler/src/forks/Scheduler.js#L550-L580)

```js
let schedulePerformWorkUntilDeadline;
if (typeof localSetImmediate === "function") {
  // Node.js and old IE.
  // There's a few reasons for why we prefer setImmediate.
  //
  // Unlike MessageChannel, it doesn't prevent a Node.js process from exiting.
  // (Even though this is a DOM fork of the Scheduler, you could get here
  // with a mix of Node.js 15+, which has a MessageChannel, and jsdom.)
  // https://github.com/facebook/react/issues/20756
  //
  // But also, it runs earlier which is the semantic we want.
  // If other browsers ever implement it, it's better to use it.
  // Although both of these would be inferior to native scheduling.
  schedulePerformWorkUntilDeadline = () => {
    localSetImmediate(performWorkUntilDeadline);
  };
} else if (typeof MessageChannel !== "undefined") {
  // DOM and Worker environments.
  // We prefer MessageChannel because of the 4ms setTimeout clamping.
  const channel = new MessageChannel();
  const port = channel.port2;
  channel.port1.onmessage = performWorkUntilDeadline;
  schedulePerformWorkUntilDeadline = () => {
    port.postMessage(null);
  };
} else {
  // We should only fallback here in non-browser environments.
  schedulePerformWorkUntilDeadline = () => {
    localSetTimeout(performWorkUntilDeadline, 0);
  };
}
```

Here we can see the 2 different fallbacks of `setImmediate()`, with MessageChannel and setTimeout.

### 2.3 Priority Queue

[Priority Queue](https://en.wikipedia.org/wiki/Priority_queue) is a common data structure for scheduling. I suggest you try to be able to [create a priority queue in JavaScript by yourself](https://bigfrontend.dev/problem/create-a-priority-queue-in-JavaScript)

It perfectly suits the needs in React. Since events of different kinds of priorities come in, we need to quickly find one with the highest priority to process.

React implements Priority Queue with minheap, you can find the [source code here](https://github.com/facebook/react/blob/main/packages/scheduler/src/SchedulerMinHeap.js).

## 3. Call stack of workLoopConcurrent

Now, let's take a look at how `workLoopConcurrent` is called.

![](/static/callstack-workloopconcurrent.png)

All the code is in [ReactFiberWorkLoop.old.js](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.old.js), let's break it down.

We've met `ensureRootIsScheduled()` a lot of times, it is used from quite a few place, as the name implies, `ensureRootIsScheduled()` schedules a task for React to do the updates if there are any.

Notice that it doesn't call `performConcurrentWorkOnRoot()` directly but treats it as a callback by `scheduleCallback(priority, callback)`.

`scheduleCallback()` is an api in [Scheduler](https://github.com/facebook/react/blob/main/packages/scheduler/src/forks/Scheduler.js).

We'll dive into the scheduler soon, but for now, just keep in mind that Scheduler will run the task at the right time.

### 3.1 performConcurrentWorkOnRoot() returns a closure of itself if interrupted.

See that performConcurrentWorkOnRoot() returns differently based on the progress?

If `shouldYield()` is true, workLoopConcurrent will break, which results in incomplete update(RootInComplete), performConcurrentWorkOnRoot() will return `performConcurrentWorkOnRoot.bind(null, root)`. ([code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L1011))

If it is complete, then it returns null.

You might wonder if some task is interrupted by `shouldYield()`, how would it resume? Yes, this is the answer. Scheduler will look at the return value of task callback, the return value is kind of rescheduling.

We'll cover this soon.

## 4. Scheduler

Finally, we are in the world of Scheduler. Don't be scared, I was scared at the beginning but soon to realize I don't need to .

Message Queue is is a way to handle out control, Scheduler does exactly like that.

`scheduleCallback()` mentioned above is [unstable_scheduleCallback](https://github.com/facebook/react/blob/main/packages/scheduler/src/forks/Scheduler.js#L308) in Scheduler world.

### 4.1 scheduleCallback() - Scheduler schedules tasks by exipriationTime

In order for Scheduler to schedule tasks, it first need to store tasks with their priorities being tagged.

This is done by Priority Queue which we've already covered as background knowledge.

It uses `expirationTime` to reflect priority. This is fair, sooner it expires, sooner we have to process it. Below is the code inside of `scheduleCallback()` where a task is created

```js
var currentTime = getCurrentTime();

var startTime;
if (typeof options === "object" && options !== null) {
  var delay = options.delay;
  if (typeof delay === "number" && delay > 0) {
    startTime = currentTime + delay;
  } else {
    startTime = currentTime;
  }
} else {
  startTime = currentTime;
}

var timeout;
switch (priorityLevel) {
  case ImmediatePriority:
    timeout = IMMEDIATE_PRIORITY_TIMEOUT;
    break;
  case UserBlockingPriority:
    timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
    break;
  case IdlePriority:
    timeout = IDLE_PRIORITY_TIMEOUT;
    break;
  case LowPriority:
    timeout = LOW_PRIORITY_TIMEOUT;
    break;
  case NormalPriority:
  default:
    timeout = NORMAL_PRIORITY_TIMEOUT;
    break;
}

var expirationTime = startTime + timeout;

var newTask = {
  id: taskIdCounter++,
  callback,
  priorityLevel,
  startTime,
  expirationTime,
  sortIndex: -1,
};
```

The code is very straightforward, for each priority we have different timeout, they are defined [here](https://github.com/facebook/react/blob/main/packages/scheduler/src/forks/Scheduler.js#L63)

```js
// Times out immediately
var IMMEDIATE_PRIORITY_TIMEOUT = -1;
// Eventually times out
var USER_BLOCKING_PRIORITY_TIMEOUT = 250;
var NORMAL_PRIORITY_TIMEOUT = 5000;
var LOW_PRIORITY_TIMEOUT = 10000;
// Never times out
var IDLE_PRIORITY_TIMEOUT = maxSigned31BitInt;
```

So default it has 5 seconds timeout, and for user blocking it has 250ms. We'll soon see some examples of these priorities.

Task is created, now it is time to put it in the Priority Queue.

```js
if (startTime > currentTime) {
  // This is a delayed task.
  newTask.sortIndex = startTime;
  push(timerQueue, newTask);
  if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
    // All tasks are delayed, and this is the task with the earliest delay.
    if (isHostTimeoutScheduled) {
      // Cancel an existing timeout.
      cancelHostTimeout();
    } else {
      isHostTimeoutScheduled = true;
    }
    // Schedule a timeout.
    requestHostTimeout(handleTimeout, startTime - currentTime);
  }
} else {
  newTask.sortIndex = expirationTime;
  push(taskQueue, newTask);
  if (enableProfiling) {
    markTaskStart(newTask, currentTime);
    newTask.isQueued = true;
  }
  // Schedule a host callback, if needed. If we're already performing work,
  // wait until the next time we yield.
  if (!isHostCallbackScheduled && !isPerformingWork) {
    isHostCallbackScheduled = true;
    requestHostCallback(flushWork);
  }
}
```

Oh right, when schedule a task, it could have a delay option, like `setTimeout()`. Let's keep it aside and come back later.

So just focus on the `else` branch. We can see two important calls

1. `push(taskQueue, newTask);` - add the task into the queue, this is just the priority queue API, I'll just skip.
2. `requestHostCallback(flushWork)` - process them!

`requestHostCallback(flushWork)` is necessary, because Scheduler is host agnostic, it should be just some independent blackbox which could be run on any host, so it needs to be 'requested'.

### 4.2 `requestHostCallback()`

```js
function requestHostCallback(callback) {
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    schedulePerformWorkUntilDeadline();
  }
}
const performWorkUntilDeadline = () => {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    // Keep track of the start time so we can measure how long the main thread
    // has been blocked.
    startTime = currentTime;
    const hasTimeRemaining = true;

    // If a scheduler task throws, exit the current browser task so the
    // error can be observed.
    //
    // Intentionally not using a try-catch, since that makes some debugging
    // techniques harder. Instead, if `scheduledHostCallback` errors, then
    // `hasMoreWork` will remain true, and we'll continue the work loop.
    let hasMoreWork = true;
    try {
      hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
    } finally {
      if (hasMoreWork) {
        // If there's more work, schedule the next message event at the end
        // of the preceding one.
        schedulePerformWorkUntilDeadline();
      } else {
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      }
    }
  } else {
    isMessageLoopRunning = false;
  }
  // Yielding to the browser will give it a chance to paint, so we can
  // reset this.
  needsPaint = false;
};
```

As mentioned in 2.2 `schedulePerformWorkUntilDeadline()` is just a wrapper on `performWorkUntilDeadline()`.

`scheduledHostCallback` is set in `requestHostCallback()` and called right away in `performWorkUntilDeadline()`, this is to give main thread a chance to render.

Ignoring some details, here is the most important line. `hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime)`, which means `flushWork()` will be called with `(true, currentTime)`.

> I don't know why it is hardcoded as true here. maybe because of refactoring.

### 4.3 flushWork()

```js
try {
  // No catch in prod code path.
  return workLoop(hasTimeRemaining, initialTime);
} finally {
  //
}
```

flushWork just wraps up `workLoop()`

### 4.4 workLoop() - the core of Scheduler

As `workLoopConcurrent()` in reconciliation, `workLoop()` is the core of Scheuler.
They have similar name because they have similar process.

```js
if (
  currentTask.expirationTime > currentTime &&
  (!hasTimeRemaining || shouldYieldToHost())
) {
  // This currentTask hasn't expired, and we've reached the deadline.
  break;
}
```

Just as `workLoopConcurrent()`, `shouldYieldToHost()` is checked here. We'll cover it later.

```js
const callback = currentTask.callback;
if (typeof callback === "function") {
  currentTask.callback = null;
  currentPriorityLevel = currentTask.priorityLevel;
  const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
  const continuationCallback = callback(didUserCallbackTimeout);
  currentTime = getCurrentTime();
  if (typeof continuationCallback === "function") {
    currentTask.callback = continuationCallback;
  } else {
    if (currentTask === peek(taskQueue)) {
      pop(taskQueue);
    }
  }
  advanceTimers(currentTime);
} else {
  pop(taskQueue);
}
```

Let's break it down

`currentTask.callback`, which is actually `performConcurrentWorkOnRoot()` in this case.

```js
const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
const continuationCallback = callback(didUserCallbackTimeout);
```

It is called with a flag to indicate if it is expired or not.

`performConcurrentWorkOnRoot()` will fall back on sync mode if timeout ([code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L920-L932))

```js
const shouldTimeSlice =
  !includesBlockingLane(root, lanes) &&
  !includesExpiredLane(root, lanes) &&
  (disableSchedulerTimeoutInWorkLoop || !didTimeout);
let exitStatus = shouldTimeSlice
  ? renderRootConcurrent(root, lanes)
  : renderRootSync(root, lanes);
```

Ok, back to `workLoop()`

```js
if (typeof continuationCallback === "function") {
  currentTask.callback = continuationCallback;
} else {
  if (currentTask === peek(taskQueue)) {
    pop(taskQueue);
  }
}
```

Important here, we can see the the task is only popped when the return value of callback is not function.
If it is function, it will just update the callback of the task, since it is not popped, next tick of workLoop() would result in the same task again.

This means **if the return value of this callback is a function, it means this task is not done, we should work on it again**, dots connect here for 3.2.

```js
advanceTimers(currentTime);
```

This is for delayed tasks, we'll come back later.

### 4.5 how `shouldYield()` work?

[source](https://github.com/facebook/react/blob/main/packages/scheduler/src/forks/Scheduler.js#L440)

```js
function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameInterval) {
    // The main thread has only been blocked for a really short amount of time;
    // smaller than a single frame. Don't yield yet.
    return false;
  }

  // The main thread has been blocked for a non-negligible amount of time. We
  // may want to yield control of the main thread, so the browser can perform
  // high priority tasks. The main ones are painting and user input. If there's
  // a pending paint or a pending input, then we should yield. But if there's
  // neither, then we can yield less often while remaining responsive. We'll
  // eventually yield regardless, since there could be a pending paint that
  // wasn't accompanied by a call to `requestPaint`, or other main thread tasks
  // like network events.
  if (enableIsInputPending) {
    if (needsPaint) {
      // There's a pending paint (signaled by `requestPaint`). Yield now.
      return true;
    }
    if (timeElapsed < continuousInputInterval) {
      // We haven't blocked the thread for that long. Only yield if there's a
      // pending discrete input (e.g. click). It's OK if there's pending
      // continuous input (e.g. mouseover).
      if (isInputPending !== null) {
        return isInputPending();
      }
    } else if (timeElapsed < maxInterval) {
      // Yield if there's either a pending discrete or continuous input.
      if (isInputPending !== null) {
        return isInputPending(continuousOptions);
      }
    } else {
      // We've blocked the thread for a long time. Even if there's no pending
      // input, there may be some other scheduled work that we don't know about,
      // like a network event. Yield now.
      return true;
    }
  }

  // `isInputPending` isn't available. Yield now.
  return true;
}
```

It is not complicated actually, the comments explains everything. The very basic lines are as below

```js
const timeElapsed = getCurrentTime() - startTime;
if (timeElapsed < frameInterval) {
  // The main thread has only been blocked for a really short amount of time;
  // smaller than a single frame. Don't yield yet.
  return false;
}
return true;
```

So each task is given 5ms (`frameInterval`), if passed it, then should yield.

Notice that this is for running the `task` in Scheduler, not for each `performUnitOfWork()`, we can see that `startTime` is only set in `performWorkUntilDeadline()`, which means it is reset for each `flushWork()`, which means if multiple tasks could be processed in on `flushWork()`, there is no yield in between.

### 5 Summary

Phew, this is a lot. let's draw an overall diagram.

![](/static/scheduler-all.png)

Still a few missing parts, but we've made a huge progress. It is already a big diagram to digest, let's put other things into next episode, including

1. delayed task in scheduler
2. how priority is determined and examples

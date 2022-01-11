---
layout: post
title: "How does React bailout work in reconciliation - React Source Code Walkthrough 13"
date: 2022-01-07 18:21:10 +0900
categories: React
---

> You can watch my [Youtube video explanation](https://www.youtube.com/watch?v=LfwMlGjiaW0) for this post

## Demo

Open this [demo link](/demos/react/how-bailout-works/index.html), there is the famous button which increments when clicked.

Open the dev console you can filter the logs of `render component`.

![](/static/bailout-1.png)

You can see the React code in the Elements tab.

![](/static/bailout-2.png)

The structure is simple:

```
<A>
  <B>
    <C>
      <button/>
      <D/>
    </C>
  </B>
  <E>
    <F/>
  </E>
</A>

```

Now click the button, as we talked before in our video series, `setState` actually triggers reconciliation from the root, theoretically all the components should be rerendered, but we only see rerender for C and D

![](/static/bailout-3.png)

## lanes & childlanes

Clear the filter in dev console, you should be able to see the logs I've already put there, you can click them to see the source code.

![](/static/bailout-4.png)

We can see a bunch of setting of `lanes` and `childLanes` after button is clicked.

In `dispatchSetState()` which is the `setCount()` in our react code, we can find the call of `scheduleUpdateOnFiber()` ([source](https://github.com/facebook/react/blob/ceee524a8f45b97c5fa9861aec3f36161495d2e1/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L460))

```js
export function scheduleUpdateOnFiber(
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
): FiberRoot | null {
  checkForNestedUpdates();

  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  if (root === null) {
    return null;
  }

  // Mark that the root has a pending update.
  markRootUpdated(root, lane, eventTime);

  ...
}
```

Yeah, we've already found it, `markUpdateLaneFromFiberToRoot()`. ([source](https://github.com/facebook/react/blob/ceee524a8f45b97c5fa9861aec3f36161495d2e1/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L566))

it does two things

1. set `lanes` of target fiber, to mark it has work to do
2. set `childLanes` of all ancestor fibers, to mark that its children has work to do.

now if we are to draw a fiber graph after clicking the button, and including `lanes` and `childLanes`, it would be like this (first number is `lanes`)

![](/static/bailout-5.png)

## performUnitOfWork()

`scheduleUpdateOnFiber()` would schedule a reconciliation callback through `ensureRootIsScheduled()`, which simply speaking, keep running `performUnitOfWork()` on all the fiber nodes. ([source](https://github.com/facebook/react/blob/ceee524a8f45b97c5fa9861aec3f36161495d2e1/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L1663-L1668))

```js
function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

`shouldYield()` is another topic about expiration which we will cover in the future. For now let's just focus on `performUnitOfWork()`.

In it , `beginWork()` has the real logic of checking if we can stop earlier(bailout) or not. ([source](https://github.com/facebook/react/blob/ceee524a8f45b97c5fa9861aec3f36161495d2e1/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L3676))

```js
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (
      oldProps !== newProps ||
      hasLegacyContextChanged()
    ) {
      // If props or context changed, mark the fiber as having performed work.
      // This may be unset if the props are determined to be equal later (memo).
      didReceiveUpdate = true;
    } else {
      // Neither props nor legacy context changes. Check if there's a pending
      // update or context change.
      const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
        current,
        renderLanes,
      );
      if (
        !hasScheduledUpdateOrContext &&
        // If this is the second pass of an error or suspense boundary, there
        // may not be work scheduled on `current`, so we check for this flag.
        (workInProgress.flags & DidCapture) === NoFlags
      ) {
        // No pending updates or context. Bail out now.
        didReceiveUpdate = false;
        return attemptEarlyBailoutIfNoScheduledUpdate(
          current,
          workInProgress,
          renderLanes,
        );
      }
      ...
    }
  } else {
    ...
  }

  workInProgress.lanes = NoLanes;

  switch (workInProgress.tag) {
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    ...
  }

}
```

just from the code we can roughly know what is being done here

1. if props and context changes, then we should continue `didReceiveUpdate = true`
2. if not, then we check if there is scheduled updates by `checkScheduledUpdateOrContext()`
3. if no scheduled updates, then we try to bailout by `attemptEarlyBailoutIfNoScheduledUpdate`
4. when update is needed, `updateFunctionComponent()` is called for functional components

One thing to notice is that the return value of `beginWork()` decides the next step of `performUnitOfWork()`. If it is null, meaning we should stop and finish. ([source](https://github.com/facebook/react/blob/ceee524a8f45b97c5fa9861aec3f36161495d2e1/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L1670))

`checkScheduledUpdateOrContext()` is simple, just checks `lanes`

```js
function checkScheduledUpdateOrContext(
  current: Fiber,
  renderLanes: Lanes,
): boolean {
  const updateLanes = current.lanes;
  if (includesSomeLane(updateLanes, renderLanes)) {
    return true;
  }
  ...
}

```

in `checkScheduledUpdateOrContext()` , `bailoutOnAlreadyFinishedWork()` is called, and `childLanes` is checked. ([source](https://github.com/facebook/react/blob/f2a59df48bec2352a4dd3b5415282a4ea240a4f8/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L3332))

So things are clear now.

1. basically React goes to every fider, from root to all the fibers
2. but if some fibers has no props changes, no context change, and both `lanes` and `childLanes` as 0, it bails out

Go back to our dev console, you can understand why A B E F are not rerendered.

A and B: try to bailout in `updateFunctionComponent()` ([source](https://github.com/facebook/react/blob/f2a59df48bec2352a4dd3b5415282a4ea240a4f8/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L1041)) since no update found, so no rerender. But their children C has work to do, so continue to C

E: bailout at `beginWork()`

F: since bailout at E, F is not checked at all.

## Wait, why D is rerenderd?

Good question.

This is because `<D/>` is in `C` which means when `C` is rerendered, it creates a new element of D, and leads to props change.

Let's explain it in more details.

in `beginWork()` if some work on `C` is found, `updateFunctionComponent()` is triggered since `C` is functional component.

To update a functional component, first we execute (rerender) it to get the new element and then reconcileChilren. ([source](https://github.com/facebook/react/blob/f2a59df48bec2352a4dd3b5415282a4ea240a4f8/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L1025))

```js
nextChildren = renderWithHooks(
  current,
  workInProgress,
  Component,
  nextProps,
  context,
  renderLanes
);
reconcileChildren(current, workInProgress, nextChildren, renderLanes);
```

In our case, children is an array of `button` and `D`, finally it goes to `reconcileChildrenArray()`. ([source](https://github.com/facebook/react/blob/c1220ebdde506de91c8b9693b5cb67ac710c8c89/packages/react-reconciler/src/ReactChildFiber.old.js#L750))

In it, we can see the code of updating the new fiber array.

```js
for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
  const newFiber = updateSlot(
    returnFiber,
    oldFiber,
    newChildren[newIdx],
    lanes,
  );
  ...
```

Drill down to `updateSlot()` then we go to `updateElement()` ([source](https://github.com/facebook/react/blob/c1220ebdde506de91c8b9693b5cb67ac710c8c89/packages/react-reconciler/src/ReactChildFiber.old.js#L390))

In `updateElement()` , below function is used to create(or reuse) a fiber

```js
function useFiber(fiber: Fiber, pendingProps: mixed): Fiber {
  // We currently set sibling to null and index to 0 here because it is easy
  // to forget to do before returning it. E.g. for the single child case.
  const clone = createWorkInProgress(fiber, pendingProps);
  clone.index = 0;
  clone.sibling = null;
  return clone;
}
```

Go to `createWorkInProgress`, we see the code where `pendingProps` is used.

```
workInProgress = createFiber(
  current.tag,
  pendingProps,
  current.key,
  current.mode,
);

// or

workInProgress.pendingProps = pendingProps;
```

Yep, `<D/>` is created every time when `C()` is executed, `pendingProps` would be different every time, though equal in value but not the same object.

So in `beginWork()`, it is treated as an update since `oldProps` is not `newProps`.

```
if (
  oldProps !== newProps ||
  hasLegacyContextChanged()
) {
  didReceiveUpdate = true;
}
```

## Move children to props leads to bailout

From the above analysis, we also understand why moving `<D/>` to children in props of `C` would leads to bailout on D.

Here is code change.

```diff
function C({children}) {
  console.log('render component C')
  const [count, setCount] = React.useState(0)
  const increment = React.useCallback(
    () => setCount(count => count + 1)
  , [])
- return <div className="component" data-name="C"><button onClick={increment}>{count}</button><D/></div>
+ return <div className="component" data-name="C"><button onClick={increment}>{count}</button>{children}</div>
}
function A() {
  console.log('render component A')
- return <div className="component" data-name="A"><B><C></C></B><E><F/></E></div>
+ return <div className="component" data-name="A"><B><C><D/></C></B><E><F/></E></div>

}
```

Go to our [second demo link](/demos/react/how-bailout-works/index2.html), again open the console and click the button, we can see D is not rerendered this time.

![](/static/bailout-6.png)

Why? Simple.

Because when `C()` is executed, `children` is passed in as an argument, which means in `createWorkInProgress()`, `pendingProps` is exactly the same, thus bailout happens.

---
layout: post
title: "How does SuspenseList work in React? | React Source Code Walkthrough 25"
date: 2022-06-15 18:21:10 +0900
categories: React
image: /static/25-SuspenseList.png
---

# 1. Demo time - What is SuspenseList?

Suspense itself will show fallbacks when not ready and reveal the contents when promises resolve, problem is that if there are multiple Suspense components, it could lead to flickering because of the order is not assured, that's why we need sorta coordinating.

[SuspenseList](https://17.reactjs.org/docs/concurrent-mode-reference.html#suspenselist) is exactly for this.

Let's first try [a demo of multiple Suspense without SuspenseList](/demos/react/suspense/multiple-suspense.html)

```jsx
<div>Hi</div>
<React.Suspense fallback={<p>loading...</p>}>
  <Child resource={resource1} />
</React.Suspense>
<React.Suspense fallback={<p>loading...</p>}>
  <Child resource={resource2} />
</React.Suspense>
<React.Suspense fallback={<p>loading...</p>}>
  <Child resource={resource3} />
</React.Suspense>
```

![](/static/multiple-suspense.gif)

We can see that the second promise is fulfilled sooner, which is kind of not cool experience.

Why don't we just use a single Suspense to hold all the `<Child/>` ? Well, This is some trade-off, using Suspense separately allows us to create better progressive experiences, we just show much as we can while they are fulfilled.

One acceptable experience would be revealing the contents from top to bottom, no matter what the resolving order is for the promises.

Let's try another [demo with SuspenseList here](/demos/react/suspense/multiple-suspense-with-suspenselist-foward.html).

> In order to try out SuspenseList, we use the experimental build

```jsx
  <div>Hi</div>
<React.SuspenseList revealOrder="forwards">
  <React.Suspense fallback={<p>loading...</p>}>
    <Child resource={resource1} />
  </React.Suspense>
  <React.Suspense fallback={<p>loading...</p>}>
    <Child resource={resource2} />
  </React.Suspense>
  <React.Suspense fallback={<p>loading...</p>}>
    <Child resource={resource3} />
  </React.Suspense>
</React.SuspenseList>
```

![](/static/suspenselist.gif)

We can see that the revealing order is kept, from top to bottom, even though the 2nd promise if fulfilled sooner.

# 2. How does SuspenseList work?

## 2.1 how to check and pass information from siblings ?

It is quite complex, let's first think about how would we implement this kind of feature by ourselves.

The core information is about **promise fulfilling order**, When a Supspense tries to reveal its content, it need some information of its siblings, including the promise state of others, and the order of itself, basically meaning it need extra info to decide to reveal or not.

Because of the tree strucuture of fiber, we are not able to share some information to siblings, the only way it through ancestors, which means we basically need some Context for more control.

In [previous post about Suspense](/react/2022/04/02/suspense-in-concurrent-mode-1-reconciling.html#lets-first-see-how-suspense-component-renders-itself), we have this piece of code in the rendering of Suspense

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

`showFallback` is determined not only by checking didSuspend of Suspense itself, but also by checking `shouldRemainOnFallback()`, this seems to be the context we are talking about.

## 2.2 shouldRemainOnFallback()

```js
// TODO: Probably should inline this back
function shouldRemainOnFallback(
  suspenseContext: SuspenseContext,
  current: null | Fiber,
  workInProgress: Fiber,
  renderLanes: Lanes
) {
  // If we're already showing a fallback, there are cases where we need to
  // remain on that fallback regardless of whether the content has resolved.
  // For example, SuspenseList coordinates when nested content appears.
  if (current !== null) {
    const suspenseState: SuspenseState = current.memoizedState;
    if (suspenseState === null) {
      // Currently showing content. Don't hide it, even if ForceSuspenseFallback
      // is true. More precise name might be "ForceRemainSuspenseFallback".
      // Note: This is a factoring smell. Can't remain on a fallback if there's
      // no fallback to remain on.
      return false;
    }
  }

  // Not currently showing content. Consult the Suspense context.
  return hasSuspenseContext(
    suspenseContext,
    (ForceSuspenseFallback: SuspenseContext)
  );
}
```

We can see from the comments, SuspenseList is explicitly mentioned. The first branch is basically say if it has already revealed, then keep showing the content.

We can see that **if ForceSuspenseFallback is in suspenseContext, then even if promise is fulfilled, fallback should still be displayed**.

## 2.3 SuspenseContext and ReactFiberStack

SuspenseContext is based on ReactFiberStack, there are a few other Context with the same implementation.

From the [source code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberSuspenseContext.new.js), SuspenseContext is something to track the information of Suspense along the path during the reconciliation, also `ForceSuspenseFallback` is just a flag of number.

```js
// ForceSuspenseFallback can be used by SuspenseList to force newly added
// items into their fallback state during one of the render passes.
export const ForceSuspenseFallback: ShallowSuspenseContext = 0b10;

export function addSubtreeSuspenseContext(
  parentContext: SuspenseContext,
  subtreeContext: SubtreeSuspenseContext
): SuspenseContext {
  return parentContext | subtreeContext;
}

export function pushSuspenseContext(
  fiber: Fiber,
  newContext: SuspenseContext
): void {
  push(suspenseStackCursor, newContext, fiber);
}

export function popSuspenseContext(fiber: Fiber): void {
  pop(suspenseStackCursor, fiber);
}
```

Let's see what is `suspenseStackCursor`.

```js
export const suspenseStackCursor: StackCursor<SuspenseContext> = createCursor(
  DefaultSuspenseContext
);
```

The secret lies in `ReactFiberStack`. ([code](https://github.com/facebook/react/blob/229c86af07302d40b70c41de18106f80fe89836c/packages/react-reconciler/src/ReactFiberStack.new.js#L59))

```js
const valueStack: Array<any> = [];

let index = -1;

function createCursor<T>(defaultValue: T): StackCursor<T> {
  return {
    current: defaultValue,
  };
}

function isEmpty(): boolean {
  return index === -1;
}

function pop<T>(cursor: StackCursor<T>, fiber: Fiber): void {
  cursor.current = valueStack[index];
  valueStack[index] = null;
  index--;
}

function push<T>(cursor: StackCursor<T>, value: T, fiber: Fiber): void {
  index++;
  valueStack[index] = cursor.current;
  cursor.current = value;
}

export { createCursor, isEmpty, pop, push };
```

`.current` points to the latest value, `valueStack` holds all the previous value so that `.current` could be set in `pop()`. Notice that there is only one `valueStack`, meaning all kind of cursors will use this same valueStack, so the `push()` and `pop()` must be exactly matched to avoid mismatch of values.

So I guess the logic would be like

1. `push()` when `beginWork()` on a fiber
2. `pop()` when `completeWork()` of a fiber

Let's see when these two functions are called.

## 2.4 when are pushSuspenseContext() called

`pushSuspenseContext()` is called in

1. `updateSuspenseComponent()` ([code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L2047))

2. `updateSuspenseListComponent()` ([code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3042))

3. `attemptEarlyBailoutIfNoScheduledUpdate()`

The first 2 are pretty straightforward, the 3rd one is some extra internal improvement, we can skip for now.

`popSuspenseContext()` is called in

1. `completeWork()` ([code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L846))
2. `unwindWork()`
3. `unwindInterruptedWork()`

Again, the 1st one is pretty straightforward.

We are going to dive into details about above timings.

## 2.5 SuspenseContext in updateSuspenseComponent()

```js
let suspenseContext: SuspenseContext = suspenseStackCursor.current;
suspenseContext = setDefaultShallowSuspenseContext(suspenseContext);

pushSuspenseContext(workInProgress, suspenseContext);
```

```js
// The Suspense Context is split into two parts. The lower bits is
// inherited deeply down the subtree. The upper bits only affect
// this immediate suspense boundary and gets reset each new
// boundary or suspense list.
const SubtreeSuspenseContextMask: SuspenseContext = 0b01;

// ForceSuspenseFallback can be used by SuspenseList to force newly added
// items into their fallback state during one of the render passes.
export const ForceSuspenseFallback: ShallowSuspenseContext = 0b10;

export function setDefaultShallowSuspenseContext(
  parentContext: SuspenseContext
): SuspenseContext {
  return parentContext & SubtreeSuspenseContextMask;
}
```

We can see that it only keeps the the lower bits -> which is for the subtree. `ForceSuspenseFallback` is the higher bit, meaning it only works inside of this current fiber.

### 2.6 How to do two passes?

Allow me to insert some background knowledge here. When we talked about [Reconciliation in Suspense]({% post_url 2022-04-02-suspense-in-concurrent-mode-1-reconciling %}), we mentioned the technique of **two passes** rendering in how Suspense handles exception.

1. renders Suspense, nothing wrong, go down to content
2. catch the exception, go back to the Suspense boundary, update some flags
3. renders Suspense again, goes to fallback because of the flags.

This is an example of how we can do branching logic inside of the tree, we can generalize the approach here if we want to do something similar.

1. create a special component for some branching logic, which holds special state
2. this component does something different base on the differnet state it has, it could do anything, for example
   - update context values
   - interrupt reconciliation process
   - (basically anything because it works like a gateway)

### 2.7 Let's review the traverse algorithm again

I'll paste the code snippet from [How does React traverse Fiber tree internally?]({% post_url 2022-01-16-fiber-traversal-in-react %}).

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

Basically it means

1. for each node we have two phase, entering (begin) and exiting (complete), just like the DOM events which has capturing phase and bubbling phase.
2. for `begin`, if return `null`, meaning there is no more work inside, so start to `complete`.
3. for `complete`, if there is sibling, will `begin` on sibling
4. Also the global `workInProgress` will be reconciled endlessly

Base on above logic, we are able to answer following questions.

**how to keep rendering a component forever?**

In completeWork, ust don't go to parent `.return`, and set the workInProgress to itself.

**how to render the component to n times?**

You can use a state to keep the render count in the component. And in `completeWork()`, check the count, if it doesn't exceed the maxium, repeat the answer to previous question.

**How to collect info from children and pass it on to other children?**

1. first we can render all the children, and expose the necessary info the fibers
2. we need also a way to interrupt the rendering, so that the control can go back to the component. Otherwise rendering just goes to the end and DOM is commited.
3. after being interruped, we can now traverse through the children again and collect the info. This is just traversing to collect, not for rendering.
4. update the context with the info we need, reconcile the child fibers again.

For SuspenseList, it is more complext because of the ordering info. Please get familiar with above knowledge and continue reading .

## 2.8 SuspenseContext in `updateSuspenseListComponent()`

```js
let suspenseContext: SuspenseContext = suspenseStackCursor.current;

const shouldForceFallback = hasSuspenseContext(
  suspenseContext,
  (ForceSuspenseFallback: SuspenseContext)
);

if (shouldForceFallback) {
  suspenseContext = setShallowSuspenseContext(
    suspenseContext,
    ForceSuspenseFallback
  );
  workInProgress.flags |= DidCapture;
} else {
  const didSuspendBefore =
    current !== null && (current.flags & DidCapture) !== NoFlags;
  if (didSuspendBefore) {
    // If we previously forced a fallback, we need to schedule work
    // on any nested boundaries to let them know to try to render
    // again. This is the same as context updating.
    propagateSuspenseContextChange(
      workInProgress,
      workInProgress.child,
      renderLanes
    );
  }
  suspenseContext = setDefaultShallowSuspenseContext(suspenseContext);
}
pushSuspenseContext(workInProgress, suspenseContext);
```

From the code we can see that SuspenseList sets `ForceSuspenseFallback` on its parent context.

But wait? Shouldn't SuspenseList be the place to initialize all the coorinating logic ? Where is the true logic that it adds `ForceSuspenseFallback` based on its descendant?

Actually the question is answered if we read a little further inside of this function.

```js
if ((workInProgress.mode & ConcurrentMode) === NoMode) {
  // In legacy mode, SuspenseList doesn't work so we just
  // use make it a noop by treating it as the default revealOrder.
  workInProgress.memoizedState = null;
} else {
  switch (revealOrder) {
    case "forwards": {
      ...
      break;
    }
    case "backwards": {
      ...
      break;
    }
    case "together": {
      ...
      break;
    }
    default: {
      // The default reveal order is the same as not having
      // a boundary.
      workInProgress.memoizedState = null;
    }
  }
}
return workInProgress.child;
```

OK, let's focus on `reveal="fowards"`, which our demo uses

```js
case "forwards": {
  const lastContentRow = findLastContentRow(workInProgress.child);
  let tail;
  if (lastContentRow === null) {
    // The whole list is part of the tail.
    // TODO: We could fast path by just rendering the tail now.
    tail = workInProgress.child;
    workInProgress.child = null;
  } else {
    // Disconnect the tail rows after the content row.
    // We're going to render them separately later.
    tail = lastContentRow.sibling;
    lastContentRow.sibling = null;
  }
  initSuspenseListRenderState(
    workInProgress,
    false, // isBackwards
    tail,
    lastContentRow,
    tailMode
  );
  break;
}
```

First it searches in the child list and find the last row which already reveals content, by `findFirstSuspended()`. Because Suspense might be deep in the tree. `findFirstSuspended()` recursively find if there exists Suspense or SuspenseList that has supended.

```js
function findLastContentRow(firstChild: null | Fiber): null | Fiber {
  // This is going to find the last row among these children that is already
  // showing content on the screen, as opposed to being in fallback state or
  // new. If a row has multiple Suspense boundaries, any of them being in the
  // fallback state, counts as the whole row being in a fallback state.
  // Note that the "rows" will be workInProgress, but any nested children
  // will still be current since we haven't rendered them yet. The mounted
  // order may not be the same as the new order. We use the new order.
  let row = firstChild;
  let lastContentRow: null | Fiber = null;
  while (row !== null) {
    const currentRow = row.alternate;
    // New rows can't be content rows.
    if (currentRow !== null && findFirstSuspended(currentRow) === null) {
      lastContentRow = row;
    }
    row = row.sibling;
  }
  return lastContentRow;
}
```

So what is point of finding `lastContentRow`? The folowing code is important

```js
if (lastContentRow === null) {
  // The whole list is part of the tail.
  // TODO: We could fast path by just rendering the tail now.
  tail = workInProgress.child;
  workInProgress.child = null;
} else {
  // Disconnect the tail rows after the content row.
  // We're going to render them separately later.
  tail = lastContentRow.sibling;
  lastContentRow.sibling = null;
}
```

`tail` means the fallback list, more accurately, it should be the start of fallbacks

`lastContentRow === null` means all are fallbacks, so tail is set to the first child, other wise, tail is set to the next sibliing.

So basically, `SuspenseList` tries to split the children into two list, one is content that is already render, the other one is the fallcks. Notice that the content in the tail is still tail, since it only search for the first fallback.

```
Content Content Content Fallback Fallback Content Fallback Content
                        ^tail
```

More interestingly is that if all are fallbacks, `workInProgress.child` is set to null, recall the algorithm mentioned in previous section, `null` means no going deeper into children, `completeWork()` kicks in on SuspenseList right away.

if there is content row, `lastContentRow.sibling = null;` means

1. it is split into two list, content list and fallback list
2. when content list is completed, by default React should to sibling, which should be fallback list, but it is disconnected meaning, `completeWork()` kicks in on SuspenseList.

We can see here that **Suspended suspenses in children will only be reconciled after completing SuspenseList** -> this is quite important.

Let's carry on, below we can see that kind of state is stored in SuspenseList.

```js
initSuspenseListRenderState(
  workInProgress,
  false, // isBackwards
  tail,
  lastContentRow,
  tailMode
);
```

```js
function initSuspenseListRenderState(
  workInProgress: Fiber,
  isBackwards: boolean,
  tail: null | Fiber,
  lastContentRow: null | Fiber,
  tailMode: SuspenseListTailMode
): void {
  const renderState: null | SuspenseListRenderState =
    workInProgress.memoizedState;
  if (renderState === null) {
    workInProgress.memoizedState = ({
      isBackwards: isBackwards,
      rendering: null,
      renderingStartTime: 0,
      last: lastContentRow,
      tail: tail,
      tailMode: tailMode,
    }: SuspenseListRenderState);
  } else {
    // We can reuse the existing object from previous renders.
    renderState.isBackwards = isBackwards;
    renderState.rendering = null;
    renderState.renderingStartTime = 0;
    renderState.last = lastContentRow;
    renderState.tail = tail;
    renderState.tailMode = tailMode;
  }
}
```

`memoizedState` on SuspenseList holds the configuration of how it should be rendered. `rendering` seems to be the target row that's needs to be revealed next.

We'll forget about other properties for now since they are just variations. If we know how `forwards` works, we know all the rest.

## 2.9 Magic lies in `completeWork()`

[code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L846)

```js
case SuspenseListComponent: {
  popSuspenseContext(workInProgress);

  const renderState: null | SuspenseListRenderState =
    workInProgress.memoizedState;

  if (renderState === null) {
    // We're running in the default, "independent" mode.
    // We don't do anything in this mode.
    bubbleProperties(workInProgress);
    return null;
  }

  let didSuspendAlready = (workInProgress.flags & DidCapture) !== NoFlags;

  const renderedTail = renderState.rendering;
  if (renderedTail === null) {
    ...
    // Next we're going to render the tail.
  } else {
    // Append the rendered row to the child list.
    ...
  }

  if (renderState.tail !== null) {
    // We still have tail rows to render.
    // Pop a row.
    const next = renderState.tail;
    renderState.rendering = next;
    renderState.tail = next.sibling;
    renderState.renderingStartTime = now();
    next.sibling = null;

    // Restore the context.
    // TODO: We can probably just avoid popping it instead and only
    // setting it the first time we go from not suspended to suspended.
    let suspenseContext = suspenseStackCursor.current;
    if (didSuspendAlready) {
      console.log("push ForceSuspenseFallback");
      suspenseContext = setShallowSuspenseContext(
        suspenseContext,
        ForceSuspenseFallback
      );
    } else {
      suspenseContext = setDefaultShallowSuspenseContext(suspenseContext);
    }
    pushSuspenseContext(workInProgress, suspenseContext);
    // Do a pass over the next row.
    // Don't bubble properties in this case.
    return next;
  }
  bubbleProperties(workInProgress);
  return null;
}
```

This is a huge chunk of code that part of it is omitted. I spent quite some time trying to understand it. Good news is that we finally see how `ForceSuspenseFallback` is set here.

Let's break it down, hang on tight.

```js
popSuspenseContext(workInProgress);

const renderState: null | SuspenseListRenderState =
  workInProgress.memoizedState;

if (renderState === null) {
  // We're running in the default, "independent" mode.
  // We don't do anything in this mode.
  bubbleProperties(workInProgress);
  return null;
}

let didSuspendAlready = (workInProgress.flags & DidCapture) !== NoFlags;
```

`renderState` is the configuration, if there is nothing, then SuspenseList is just a no-op component.

`didSuspendAlready` is a local flag to tell SuspenseList to find the first Suspended Suspense.

SuspenseList also has `DidCapture` flag, ecause in `forwards`, there is revealing order required, so once SuspenseList finds the first Suspended Suspense, it will use the same promise to trigger and update.`didSuspendAlready` could be used to avoid using the upcoming promises.

```js
const renderedTail = renderState.rendering;
if (renderedTail === null) {
  // We just rendered the head.
  ....
  // Next we're going to render the tail.
} else {
  ...
}

if (renderState.tail !== null) {
    // We still have tail rows to render.
    // Pop a row.
    const next = renderState.tail;
    renderState.rendering = next;
    renderState.tail = next.sibling;
    renderState.renderingStartTime = now();
    next.sibling = null;

    // Restore the context.
    // TODO: We can probably just avoid popping it instead and only
    // setting it the first time we go from not suspended to suspended.
    let suspenseContext = suspenseStackCursor.current;
    if (didSuspendAlready) {
      suspenseContext = setShallowSuspenseContext(
        suspenseContext,
        ForceSuspenseFallback,
      );
    } else {
      suspenseContext = setDefaultShallowSuspenseContext(suspenseContext);
    }
    pushSuspenseContext(workInProgress, suspenseContext);
    // Do a pass over the next row.
    // Don't bubble properties in this case.
    return next;
  }
  bubbleProperties(workInProgress);
  return null;
```

This last piece of code is very important

1. we can see the `tail` is rolling forward by one, `renderState.tail = next.sibling`
2. old tail is isolated by `next.sibling = null`, which means when it is reconciled, `completeWork()` goes to SuspenseList again, rather than go to previous sibling.
3. the old `tail` is returned, meaning `beginWork()` will start on it.

Inside of the omitted code.

```js
if (renderedTail === null) {
  // We just rendered the head.
  if (!didSuspendAlready) {
    // This is the first pass. We need to figure out if anything is still
    // suspended in the rendered set.

    // If new content unsuspended, but there's still some content that
    // didn't. Then we need to do a second pass that forces everything
    // to keep showing their fallbacks.

    // We might be suspended if something in this render pass suspended, or
    // something in the previous committed pass suspended. Otherwise,
    // there's no chance so we can skip the expensive call to
    // findFirstSuspended.
    const cannotBeSuspended =
      renderHasNotSuspendedYet() &&
      (current === null || (current.flags & DidCapture) === NoFlags);
    if (!cannotBeSuspended) {
      let row = workInProgress.child;
      while (row !== null) {
        const suspended = findFirstSuspended(row);
        if (suspended !== null) {
          didSuspendAlready = true;
          workInProgress.flags |= DidCapture;
          cutOffTailIfNeeded(renderState, false);

          // If this is a newly suspended tree, it might not get committed as
          // part of the second pass. In that case nothing will subscribe to
          // its thenables. Instead, we'll transfer its thenables to the
          // SuspenseList so that it can retry if they resolve.
          // There might be multiple of these in the list but since we're
          // going to wait for all of them anyway, it doesn't really matter
          // which ones gets to ping. In theory we could get clever and keep
          // track of how many dependencies remain but it gets tricky because
          // in the meantime, we can add/remove/change items and dependencies.
          // We might bail out of the loop before finding any but that
          // doesn't matter since that means that the other boundaries that
          // we did find already has their listeners attached.
          const newThenables = suspended.updateQueue;
          if (newThenables !== null) {
            workInProgress.updateQueue = newThenables;
            workInProgress.flags |= Update;
          }

          // Rerender the whole list, but this time, we'll force fallbacks
          // to stay in place.
          // Reset the effect flags before doing the second pass since that's now invalid.
          // Reset the child fibers to their original state.
          workInProgress.subtreeFlags = NoFlags;
          resetChildFibers(workInProgress, renderLanes);

          // Set up the Suspense Context to force suspense and immediately
          // rerender the children.
          pushSuspenseContext(
            workInProgress,
            setShallowSuspenseContext(
              suspenseStackCursor.current,
              ForceSuspenseFallback
            )
          );
          // Don't bubble properties in this case.
          return workInProgress.child;
        }
        row = row.sibling;
      }
    }

    if (renderState.tail !== null && now() > getRenderTargetTime()) {
      // We have already passed our CPU deadline but we still have rows
      // left in the tail. We'll just give up further attempts to render
      // the main content and only render fallbacks.
      workInProgress.flags |= DidCapture;
      didSuspendAlready = true;

      cutOffTailIfNeeded(renderState, false);

      // Since nothing actually suspended, there will nothing to ping this
      // to get it started back up to attempt the next item. While in terms
      // of priority this work has the same priority as this current render,
      // it's not part of the same transition once the transition has
      // committed. If it's sync, we still want to yield so that it can be
      // painted. Conceptually, this is really the same as pinging.
      // We can use any RetryLane even if it's the one currently rendering
      // since we're leaving it behind on this node.
      workInProgress.lanes = SomeRetryLane;
    }
  } else {
    cutOffTailIfNeeded(renderState, false);
  }
  // Next we're going to render the tail.
}
```

About branch is right after we split the content list and fallback list, what it roughly does is

1. find the first suspended suspense and connect the promises
2. set the `ForceSuspenseFallback` in the SuspenseContext
3. rerender the whole list after `resetChildFibers()`, which reverts the splitting.

Why rerender the whole list again? I guess since we are reconciling, the other promises might already get fulfilled. If we don't force everything to render fallbacks at the beginning, the ordering actually is going to break for the initial state. So this rerendering makes sure SuspenseList has a clean slate to work on.

For the other branch of second pass, we don't have the need to rerender again.

```js
} else {
  // Append the rendered row to the child list.
  if (!didSuspendAlready) {
    const suspended = findFirstSuspended(renderedTail);
    if (suspended !== null) {
      workInProgress.flags |= DidCapture;
      didSuspendAlready = true;

      // Ensure we transfer the update queue to the parent so that it doesn't
      // get lost if this row ends up dropped during a second pass.
      const newThenables = suspended.updateQueue;
      if (newThenables !== null) {
        workInProgress.updateQueue = newThenables;
        workInProgress.flags |= Update;
      }

      cutOffTailIfNeeded(renderState, true);
      // This might have been modified.
      if (
        renderState.tail === null &&
        renderState.tailMode === 'hidden' &&
        !renderedTail.alternate &&
        !getIsHydrating() // We don't cut it if we're hydrating.
      ) {
        // We're done.
        bubbleProperties(workInProgress);
        return null;
      }
    } else if (
      // The time it took to render last row is greater than the remaining
      // time we have to render. So rendering one more row would likely
      // exceed it.
      now() * 2 - renderState.renderingStartTime >
        getRenderTargetTime() &&
      renderLanes !== OffscreenLane
    ) {
      // We have now passed our CPU deadline and we'll just give up further
      // attempts to render the main content and only render fallbacks.
      // The assumption is that this is usually faster.
      workInProgress.flags |= DidCapture;
      didSuspendAlready = true;

      cutOffTailIfNeeded(renderState, false);

      // Since nothing actually suspended, there will nothing to ping this
      // to get it started back up to attempt the next item. While in terms
      // of priority this work has the same priority as this current render,
      // it's not part of the same transition once the transition has
      // committed. If it's sync, we still want to yield so that it can be
      // painted. Conceptually, this is really the same as pinging.
      // We can use any RetryLane even if it's the one currently rendering
      // since we're leaving it behind on this node.
      workInProgress.lanes = SomeRetryLane;
    }
  }
  if (renderState.isBackwards) {
    // The effect list of the backwards tail will have been added
    // to the end. This breaks the guarantee that life-cycles fire in
    // sibling order but that isn't a strong guarantee promised by React.
    // Especially since these might also just pop in during future commits.
    // Append to the beginning of the list.
    renderedTail.sibling = workInProgress.child;
    workInProgress.child = renderedTail;
  } else {
    const previousSibling = renderState.last;
    if (previousSibling !== null) {
      previousSibling.sibling = renderedTail;
    } else {
      workInProgress.child = renderedTail;
    }
    renderState.last = renderedTail;
  }
}
```

Quite some details, but I'll skip for now, the idea

## 2.10 Summary

Let me try to summarize what is going on in SuspenseList:

1. when update SuspenseList, it first split the children into to list, `head` and `tail`, by searching for the last content row(not suspended suspense)
2. `head` is rendered as normal
3. SuspenseList renders the tails one by one, in `completeWork()`
   - disconnecting each from the sibling. This leads to `completeWork()` on SuspenseList every time child is completed.
   - It checks for first Suspended Suspense set up the retrylisteners on the promise, and set up `ForceSuspsensFallback` in the context, thus Suspenses coming later renders fallback even not supended
   - There are some checks if there is no tail to render, which rerender the whole list to check if rendered heads become suspended again.

Why popping the tails one by one in second phase?

I'm not sure. I guess it is to gradually move things out from the tail, just because SuspenseList are supposed to have a long list of suspenses, it could be time consuming to render all the children. For the first pass, since all are going to be fallbacks, it is fine. But for the revealing phase, it is a different story, when say the reconciliation is interrupted and later be resumed, we need to track inside of SuspenseList where it checked last time. This info needs to be done in the completeWork() of SuspenseList.

## 2.11 Illustration

Yep, above is just brain consuming to understand. I've prepared a diagram to explain.

![](/static/suspenselist-1.png)

At the beginning, we'll just start to reconcile SuspenseList.

![](/static/suspenselist-2.png)

In the initial step, no content row is found (new fibers are not count), so `tail` is set to `div`, and `child` is set to null from SuspenseList, meaning all rows are `tail`.

![](/static/suspenselist-3.png)

Since `child` is null, there no more work to do, `completeWork()` kicks in

![](/static/suspenselist-4.png)

Since this is the very initial render, there is no step of rendering the whole list, but that SuspenseList start to render the tail one by one. We can see the the fiber being rendered has `sibling` being removed.

![](/static/suspenselist-5.png)

There is nothing more on `div`, so complete.

![](/static/suspenselist-6.png)

And `completeWork()` on SuspenseList again.

![](/static/suspenselist-7.png)

Eventually `tail` is going to be `null` and the loop stops. Because the initial render doesn't have any suspended suspense. so the process ended.

### After button is clicked

![](/static/suspenselist-8.png)

The last Suspense is `lastContentRow`, so `tail` is set to its sibling which is still `null`, and reconciliation of content rows continue

![](/static/suspenselist-9.png)

`div` is rendered

![](/static/suspenselist-10.png)

Then comes the first Suspense

![](/static/suspenselist-11.png)

Suspense suspends this time, rendering fallback of `p`. The true structure is more complex with Offscreen component, I'll just used a dotted line to show that fallback is rendered

![](/static/suspenselist-12.png)

Eventually all suspenses fallback are rendered, and `completeWork()` works on `SuspenseList` again.

![](/static/suspenselist-13.png)

Since now SuspenseList is **possible to suspend**, it searchs its children and found a suspended suspense, so `DidCapture` is set, and `ForceSuspenseFallback` is set into `SuspenseContex`, also the whole list is rerendered.

![](/static/suspenselist-14.png)

It goes to next Suspense, since the ForceSuspenseFallback is there, all suspenses renders fallback without deeper check.

![](/static/suspenselist-15.png)

Eventually `completeWork()` is called again on SuspenseList, but there is no `tail` left to render, so it is done

## When second Promise is fulfilled.

![](/static/suspenselist-16.png)

The process is similar as before, first the flags and context flags are reset, `tail` is set to the first suspended supsense, and `head` is diconnected from `tail`.

![](/static/suspenselist-17.png)

`div` is worked on.

![](/static/suspenselist-18.png)

prepare to render tails

![](/static/suspenselist-19.png)

go to first Suspense

![](/static/suspenselist-20.png)

`completeWork()` starts on SuspenseList, this time since there is `tail` waiting to be rendered, SuspenseList searches its children and find the the first Suspense is suspended, so the flags are set again.

![](/static/suspenselist-21.png)

Now tail goes to 2nd Suspense, even its promise is fulfilled, it still renders fallback because of the ForceSuspenseFallback falg in suspense context.

This is how reveal order is kept.

Ok I'll skip the rest from here.

## When the first Promise is resolved.

Basically the flow is the same, just the first 2 promises are fulfilled without any exception, so they reveal their content

![](/static/suspenselist-22.png)

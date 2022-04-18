---
layout: post
title: "How Suspense works internally in Concurrent Mode 2 - Offscreen component | React Source Code Walkthrough 23"
date: 2022-04-17 18:21:10 +0900
categories: React
image: /static/offscreen-23.png
---

In previous post about [Reconciling in Suspense]({% post_url 2022-04-02-suspense-in-concurrent-mode-1-reconciling %}), we have seen how Offscreen component is used under the hood of Suspense.

In the [release note of React 18](https://reactjs.org/blog/2022/03/29/react-v18.html), React team has mentioned their intention to expose this Offscreen component.

In this episode, let's take a deeper look at how Offscreen component works.

- [Demo](#demo)
- [Data type for OffscreenComponent](#data-type-for-offscreencomponent)
- [Reconciling Offscreen Component](#reconciling-offscreen-component)
  - [How Offscreen rendering is scheduled](#how-offscreen-rendering-is-scheduled)
  - [How Offscreen rendering is done?](#how-offscreen-rendering-is-done)
  - [How is it rendered but hidden from DOM?](#how-is-it-hidden-from-dom)
  - [How Offscreen Component handles hidden => visible?](#how-offscreen-component-handles-hidden--visible)
  - [Offscreen Component decides to hide/unhide in commit phase](#offscreen-component-decides-to-hideunhide-in-commit-phase)
  - [Summary](#summary)

## Demo

In order to try out Offscreen component, we need to use a build with experiemental features on.

I've put up a [demo here](/demos/react/offscreen/), you can try it out.

```jsx
const Offscreen = React.unstable_Offscreen;
function Component() {
  const [count, setCount] = React.useState(0);
  console.log("render Component: count => ", count);
  React.useLayoutEffect(() => {
    console.log("render Component: layout effect in Component");
  }, []);
  React.useEffect(() => {
    console.log("render Component: effect in Component");
    setCount((_) => _ + 1);
  }, []);
  return <p>{count}</p>;
}

function App() {
  const [hidden, setHidden] = React.useState(true);
  console.log("render App");
  return (
    <div>
      <button onClick={() => setHidden((_) => !_)}>toggle</button>
      <Offscreen mode={hidden ? "hidden" : "visible"}>
        <Component />
      </Offscreen>
    </div>
  );
}

const rootElement = document.getElementById("container");
ReactDOM.createRoot(rootElement).render(<App />);
```

Open console, we can see some interesting things

1. when Component is invisible
   - it is still rendered
   - passive effects are run
   - layout effects are NOT run
   - renderRootConcurrent is called twice
2. when Component is visible after clicking the button
   - layout effects are run

We know that "render" means constructing or updating the intenerl React fiber tree, after it is ready, it is reflected into DOM in commit phase.

Also if we inspect the invisible state, we can see that the hidden elements are there but just hidden.

![](/static/offscreen-style-hidden.png)

So from above behaviors we can see what Offscreen does is

**somehow defer the rendering of invisible contents and hide them with CSS**

## Data type for OffscreenComponent

```js
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

export type OffscreenProps = {|
  // TODO: Pick an API before exposing the Offscreen type. I've chosen an enum
  // for now, since we might have multiple variants. For example, hiding the
  // content without changing the layout.
  //
  // Default mode is visible. Kind of a weird default for a component
  // called "Offscreen." Possible alt: <Visibility />?
  mode?: OffscreenMode | null | void,
  children?: ReactNodeList,
|};

// We use the existence of the state object as an indicator that the component
// is hidden.
export type OffscreenState = {|
  // TODO: This doesn't do anything, yet. It's always NoLanes. But eventually it
  // will represent the pending work that must be included in the render in
  // order to unhide the component.
  baseLanes: Lanes,
  cachePool: SpawnedCachePool | null,
|};

export type OffscreenInstance = {};

export type OffscreenMode =
  | "hidden"
  | "unstable-defer-without-hiding"
  | "visible";
```

1. `REACT_OFFSCREEN_TYPE` is the type of Offscreen element. it has `hidden` `visible` or `unstable-defer-without-hiding` as its mode
2. `OffscreenState` is important if it is not null, it means Offscreen is invisible.

Here is a basic example of how `createFiberFromOffscreen()` is used in Suspense.

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

Here we create a Offscreen fiber which is "visible" and wrapping Suspense children as children.

## Reconciling Offscreen Component

Since Offscreen works like a switch, it doesn't hold some references to the real host nodes, it only has updating reconciling methods. ([source](https://github.com/facebook/react/blob/8e2f9b086e7abc7a92951d264a6a5d048defd914/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L636))

```js
function updateOffscreenComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
) {
  const nextProps: OffscreenProps = workInProgress.pendingProps;
  const nextChildren = nextProps.children;

  const prevState: OffscreenState | null =
    current !== null ? current.memoizedState : null;

  if (nextProps.mode === "hidden" || enableLegacyHidden) {
    // Rendering a hidden tree.
    if ((workInProgress.mode & ConcurrentMode) === NoMode) {
      // legacy mode
      ...
    } else if (!includesSomeLane(renderLanes, OffscreenLane)) {
      // prepare to render hidden component in OffscreenLane
      ...
    } else {
      // render hidden component in OffscreenLane
      ...
    }
  } else {
    // Rendering a visible tree.
    ...
  }

  // go to children
  {
    reconcileChildren(current, workInProgress, nextChildren, renderLanes);
    return workInProgress.child;
  }
}
```

It is quite a bit of code so first let's look at the outmost if branches. With the comments above we can see that

1. even when "hidden", rendering still happens but under OffscreenLane
2. OffscreenLane is added on the fly, so there are 2 steps, one is prepare one is to render and thus the process is deferred.

```js
export const IdleLane: Lanes = /*                       */ 0b0100000000000000000000000000000;
export const OffscreenLane: Lane = /*                   */ 0b1000000000000000000000000000000;
```

OffscreenLane has the lowest piroirty, even lower than IdleLane, which is reasonable because if it is hidden, it means invisible to users, we should handle it in the end.

### How Offscreen rendering is scheduled ?

```js
if (!includesSomeLane(renderLanes, (OffscreenLane: Lane))) {
```

As we mentioned, rendering of hidden components are scheduled in OffscreeLane, so if OffscreenLane is not in current renderLanes, it means it should be scheduled now.

`spawnedCachePool` is something about Cache component, let's skip it for now.

```js
// We're hidden, and we're not rendering at Offscreen. We will bail out
// and resume this tree later.
let nextBaseLanes;
if (prevState !== null) {
  const prevBaseLanes = prevState.baseLanes;
  nextBaseLanes = mergeLanes(prevBaseLanes, renderLanes);
} else {
  nextBaseLanes = renderLanes;
}
```

It prepares the baseLanes by merging previous baseLanes with current renderLanes, this is a bit tricky.

Suppose the below case

1. we are rendering at SyncLane, the targeted fiber is under Offscreen component
2. and we bailed out at Offscreen component, means the fiber cannot be updated
3. When we continue the invisible rendering at OffscreenLane, we need to include the SyncLane

So `OffscreenState.baseLanes` is a way to store the previosly skipped working lanes. To illustrated this, I created [another demo](/demos/react/offscreen//work-left/)

```jsx
function Component({ onClick }) {
  const [count, setCount] = React.useState(0);
  console.log("visible, render Component:", count);
  return (
    <div>
      <button
        onClick={() => {
          setCount((_) => _ + 1);
          onClick();
        }}
      >
        schedule work and hide the offscreen component
      </button>
      <p>{count}</p>
    </div>
  );
}

function App() {
  const [hidden, setHidden] = React.useState(false);

  console.log("render App");
  return (
    <div>
      <button onClick={() => setHidden((_) => !_)}>
        toggle offscreen component
      </button>
      <Offscreen mode={hidden ? "hidden" : "visible"}>
        <Component
          onClick={() => {
            setHidden(true);
          }}
        />
      </Offscreen>
    </div>
  );
}
```

Open the demo and console, you can see that if we click the 2nd button,

1. `subtreeRenderLanes is set to 00000000000000000000000000000001` => event actions is on sync lane
2. `pushrenderLanes 1` => Offscreen bails out
3. `2nd render set to NoLanes 1000000000000000000000000000000` => renders again on OffscrenLane
4. `subtreeRenderLanes is set to 1000000000000000000000000000001` => it combines the first SyncLane which is skipped
5. `enough priority` => when render our `<Componenent>`, it has enough priority because it is scheduled on SyncLane, it needs to be on SyncLane.

Phew, this is a log. Let's go back to how OffscreenLane is scheduled.

```js
// Schedule this fiber to re-render at offscreen priority. Then bailout.
workInProgress.lanes = workInProgress.childLanes = laneToLanes(OffscreenLane);
```

This line is important, it set `lanes` to mark that this Offscreen component needs to be re-renderd. Recall we've covered this in [How does React bailout work in reconciliation]{% post_url 2022-01-08-how-does-bailout-work %}).

There was `markUpdateLaneFromFiberToRoot()` which helps update the `childLanes` straight to root, but there is no such call here, it is handled by some other logic. Let's continue.

```js
const nextState: OffscreenState = {
  baseLanes: nextBaseLanes,
  cachePool: spawnedCachePool,
};
workInProgress.memoizedState = nextState;
workInProgress.updateQueue = null;
// We're about to bail out, but we need to push this to the stack anyway
// to avoid a push/pop misalignment.
pushRenderLanes(workInProgress, nextBaseLanes);

return null;
```

It create `OffscreenState` and set it to this Offscreen, remember `OffscreenState` is to indicate it is invisible.

We should be familiar with the `return null`, it is to tell reconciler that bailout should happen, stop going deeper and start completing.

Inside of `completeWork()`. ([source](https://github.com/facebook/react/blob/8e2f9b086e7abc7a92951d264a6a5d048defd914/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L832))

```js
case OffscreenComponent:
case LegacyHiddenComponent: {
  popRenderLanes(workInProgress);
  var _nextState = workInProgress.memoizedState;
  var nextIsHidden = _nextState !== null;

  if (current !== null) {
    var _prevState2 = current.memoizedState;
    var prevIsHidden = _prevState2 !== null;

    if (
      prevIsHidden !== nextIsHidden && // LegacyHidden doesn't do any hiding — it only pre-renders.
      !enableLegacyHidden
    ) {
      workInProgress.flags |= Visibility;
    }
  }

  if (
    !nextIsHidden ||
    (workInProgress.mode & ConcurrentMode) === NoMode
  ) {
    bubbleProperties(workInProgress);
  } else {
    // Don't bubble properties for hidden children unless we're rendering
    // at offscreen priority.
    if (includesSomeLane(subtreeRenderLanes, OffscreenLane)) {
      bubbleProperties(workInProgress);

      {
        // Check if there was an insertion or update in the hidden subtree.
        // If so, we need to hide those nodes in the commit phase, so
        // schedule a visibility effect.
        if (workInProgress.subtreeFlags & (Placement | Update)) {
          workInProgress.flags |= Visibility;
        }
      }
    }
  }

  popTransition(workInProgress, current);
  return null;
}
```

1. it checks if `prevIsHidden !== nextIsHidden` or they are any insertion .etc in the children and set `Visibility` flag to indicate it needs update
2. `bubbleProperties()` is called

And actually `lanes` are collected and set to `childLanes` of parent fiber in this function.

```js
function bubbleProperties(completedWork) {
  var didBailout =
    completedWork.alternate !== null &&
    completedWork.alternate.child === completedWork.child;
  var newChildLanes = NoLanes;
  var subtreeFlags = NoFlags;

  if (!didBailout) {
    // Bubble up the earliest expiration time.
    if ((completedWork.mode & ProfileMode) !== NoMode) {
      // In profiling mode, resetChildExpirationTime is also used to reset
      // profiler durations.
      var actualDuration = completedWork.actualDuration;
      var treeBaseDuration = completedWork.selfBaseDuration;
      var child = completedWork.child;

      while (child !== null) {
        newChildLanes = mergeLanes(
          newChildLanes,
          mergeLanes(child.lanes, child.childLanes)
        );
        subtreeFlags |= child.subtreeFlags;
        subtreeFlags |= child.flags; // When a fiber is cloned, its actualDuration is reset to 0. This value will
        // only be updated if work is done on the fiber (i.e. it doesn't bailout).
        // When work is done, it should bubble to the parent's actualDuration. If
        // the fiber has not been cloned though, (meaning no work was done), then
        // this value will reflect the amount of time spent working on a previous
        // render. In that case it should not bubble. We determine whether it was
        // cloned by comparing the child pointer.

        actualDuration += child.actualDuration;
        treeBaseDuration += child.treeBaseDuration;
        child = child.sibling;
      }

      completedWork.actualDuration = actualDuration;
      completedWork.treeBaseDuration = treeBaseDuration;
    } else {
      var _child = completedWork.child;

      while (_child !== null) {
        newChildLanes = mergeLanes(
          newChildLanes,
          mergeLanes(_child.lanes, _child.childLanes)
        );
        subtreeFlags |= _child.subtreeFlags;
        subtreeFlags |= _child.flags; // Update the return pointer so the tree is consistent. This is a code
        // smell because it assumes the commit phase is never concurrent with
        // the render phase. Will address during refactor to alternate model.

        _child.return = completedWork;
        _child = _child.sibling;
      }
    }

    completedWork.subtreeFlags |= subtreeFlags;
  } else {
    // Bubble up the earliest expiration time.
    if ((completedWork.mode & ProfileMode) !== NoMode) {
      // In profiling mode, resetChildExpirationTime is also used to reset
      // profiler durations.
      var _treeBaseDuration = completedWork.selfBaseDuration;
      var _child2 = completedWork.child;

      while (_child2 !== null) {
        newChildLanes = mergeLanes(
          newChildLanes,
          mergeLanes(_child2.lanes, _child2.childLanes)
        ); // "Static" flags share the lifetime of the fiber/hook they belong to,
        // so we should bubble those up even during a bailout. All the other
        // flags have a lifetime only of a single render + commit, so we should
        // ignore them.

        subtreeFlags |= _child2.subtreeFlags & StaticMask;
        subtreeFlags |= _child2.flags & StaticMask;
        _treeBaseDuration += _child2.treeBaseDuration;
        _child2 = _child2.sibling;
      }

      completedWork.treeBaseDuration = _treeBaseDuration;
    } else {
      var _child3 = completedWork.child;

      while (_child3 !== null) {
        newChildLanes = mergeLanes(
          newChildLanes,
          mergeLanes(_child3.lanes, _child3.childLanes)
        ); // "Static" flags share the lifetime of the fiber/hook they belong to,
        // so we should bubble those up even during a bailout. All the other
        // flags have a lifetime only of a single render + commit, so we should
        // ignore them.

        subtreeFlags |= _child3.subtreeFlags & StaticMask;
        subtreeFlags |= _child3.flags & StaticMask; // Update the return pointer so the tree is consistent. This is a code
        // smell because it assumes the commit phase is never concurrent with
        // the render phase. Will address during refactor to alternate model.

        _child3.return = completedWork;
        _child3 = _child3.sibling;
      }
    }

    completedWork.subtreeFlags |= subtreeFlags;
  }

  completedWork.childLanes = newChildLanes;
  return didBailout;
}
```

It is also complex, but we can see there is the while loop to gather sibling flags and lanes, the last line is to reflect them on parent fiber.

Why we are doing it in `completeWork()` is reasonble, we'll traverse through the ancestor fibers here anyway so we should do it here.

Also `ensureRootIsScheduled()` is always called when one render is done, so after the 1st pass, React will try to check if there is more work to do, in 2nd pass, it finds the OffscreenLane and able get it renderred.

## How Offscreen rendering is done?

```js
// This is the second render. The surrounding visible content has already
// committed. Now we resume rendering the hidden tree.

// Rendering at offscreen, so we can clear the base lanes.
const nextState: OffscreenState = {
  baseLanes: NoLanes,
  cachePool: null,
};
workInProgress.memoizedState = nextState;
// Push the lanes that were skipped when we bailed out.
const subtreeRenderLanes =
  prevState !== null ? prevState.baseLanes : renderLanes;

pushRenderLanes(workInProgress, subtreeRenderLanes);
```

Nothing fancy here, one difference here is that there is no `return null`, so the reconciler go down to the children, that's how we see our `<Component/>` in the get rendered.

## How is it rendered but hidden from DOM?

We know that DOM manipulation is in commit phase, let's try to find the answer.

But before we jumps in, let's recall how it handles intrinsic DOM element in `completeWork()`

```js
case HostComponent: {
  popHostContext(workInProgress);
  const rootContainerInstance = getRootHostContainer();
  const type = workInProgress.type;
  ...
  const instance = createInstance(
    type,
    newProps,
    rootContainerInstance,
    currentHostContext,
    workInProgress,
  );

  appendAllChildren(instance, workInProgress, false, false);

  workInProgress.stateNode = instance;
  ...
```

I've omitted a lot of lines here, but just to see that the DOM construction is done in `completeWork()`. But for Offscreen component we don't have DOM nodes to handle.

But `appendAllChildren()` actually will skip Offscreen component and collects DOM nodes in its children, right?

Good question, remember that the rendering of invisible contents are deferred, completeWork will be called twice. In the first pass, there are no DOM elements created yet, so nothing is added, and in the second pass, all props are not changed, it won't do anything here.

The DOM is connected in commit phase, I guess because we need to hide it synchronously commit phase is the only choice, since there is no interrupting in committing phase.

> Notice this is only about "invisible state", if visible, it just normally append all children as it should be.

## How Offscreen Component handles hidden => visible?

In `updateOffscreenComponent()`.

```js
// Rendering a visible tree.
let subtreeRenderLanes;
if (prevState !== null) {
  // We're going from hidden -> visible.
  subtreeRenderLanes = mergeLanes(prevState.baseLanes, renderLanes);
  let prevCachePool = null;
  if (enableCache) {
    // If the render that spawned this one accessed the cache pool, resume
    // using the same cache. Unless the parent changed, since that means
    // there was a refresh.
    prevCachePool = prevState.cachePool;
  }

  pushTransition(workInProgress, prevCachePool, null);

  // Since we're not hidden anymore, reset the state
  workInProgress.memoizedState = null;
} else {
  // We weren't previously hidden, and we still aren't, so there's nothing
  // special to do. Need to push to the stack regardless, though, to avoid
  // a push/pop misalignment.
  subtreeRenderLanes = renderLanes;

  if (enableCache) {
    // If the render that spawned this one accessed the cache pool, resume
    // using the same cache. Unless the parent changed, since that means
    // there was a refresh.
    if (current !== null) {
      pushTransition(workInProgress, null, null);
    }
  }
}
pushRenderLanes(workInProgress, subtreeRenderLanes);
```

Nothing fancy here, just clears `memoizedState` to indicate it is visible, so magic doesn't lie here.

## Offscreen Component decides to hide/unhide in commit phase

Actually the magic is commit phase, `commitMutationEffectsOnFiber()`. ([source](https://github.com/facebook/react/blob/8e2f9b086e7abc7a92951d264a6a5d048defd914/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L1986))

```js
case OffscreenComponent: {
  var _wasHidden = current !== null && current.memoizedState !== null;
  // Before committing the children, track on the stack whether this
  // offscreen subtree was already hidden, so that we don't unmount the
  // effects again.
  var prevOffscreenSubtreeWasHidden = offscreenSubtreeWasHidden;
  offscreenSubtreeWasHidden = prevOffscreenSubtreeWasHidden || _wasHidden;
  recursivelyTraverseMutationEffects(root, finishedWork);
  offscreenSubtreeWasHidden = prevOffscreenSubtreeWasHidden;
  commitReconciliationEffects(finishedWork);

  if (flags & Visibility) {
    var _newState = finishedWork.memoizedState;

    var _isHidden = _newState !== null;

    var offscreenBoundary = finishedWork;

    {
      // TODO: This needs to run whenever there's an insertion or update
      // inside a hidden Offscreen tree.
      hideOrUnhideAllChildren(offscreenBoundary, _isHidden);
    }

    {
      if (_isHidden) {
        if (!_wasHidden) {
          if ((offscreenBoundary.mode & ConcurrentMode) !== NoMode) {
            nextEffect = offscreenBoundary;
            var offscreenChild = offscreenBoundary.child;

            while (offscreenChild !== null) {
              nextEffect = offscreenChild;
              disappearLayoutEffects_begin(offscreenChild);
              offscreenChild = offscreenChild.sibling;
            }
          }
        }
      }
    }
  }

  return;
}
```

we can see there are 2 thing done

1. `hideOrUnhideAllChildren()` if visibility changes
2. trigger layout effects when it becomes visible.

```js
function hideOrUnhideAllChildren(finishedWork, isHidden) {
  // Only hide or unhide the top-most host nodes.
  let hostSubtreeRoot = null;

  if (supportsMutation) {
    // We only have the top Fiber that was inserted but we need to recurse down its
    // children to find all the terminal nodes.
    let node: Fiber = finishedWork;
    while (true) {
      if (node.tag === HostComponent) {
        if (hostSubtreeRoot === null) {
          hostSubtreeRoot = node;
          try {
            const instance = node.stateNode;
            if (isHidden) {
              hideInstance(instance);
            } else {
              unhideInstance(node.stateNode, node.memoizedProps);
            }
          } catch (error) {
            captureCommitPhaseError(finishedWork, finishedWork.return, error);
          }
        }
      } else if (node.tag === HostText) {
        if (hostSubtreeRoot === null) {
          try {
            const instance = node.stateNode;
            if (isHidden) {
              hideTextInstance(instance);
            } else {
              unhideTextInstance(instance, node.memoizedProps);
            }
          } catch (error) {
            captureCommitPhaseError(finishedWork, finishedWork.return, error);
          }
        }
      } else if (
        (node.tag === OffscreenComponent ||
          node.tag === LegacyHiddenComponent) &&
        (node.memoizedState: OffscreenState) !== null &&
        node !== finishedWork
      ) {
        // Found a nested Offscreen component that is hidden.
        // Don't search any deeper. This tree should remain hidden.
      } else if (node.child !== null) {
        node.child.return = node;
        node = node.child;
        continue;
      }

      if (node === finishedWork) {
        return;
      }
      while (node.sibling === null) {
        if (node.return === null || node.return === finishedWork) {
          return;
        }

        if (hostSubtreeRoot === node) {
          hostSubtreeRoot = null;
        }

        node = node.return;
      }

      if (hostSubtreeRoot === node) {
        hostSubtreeRoot = null;
      }

      node.sibling.return = node.return;
      node = node.sibling;
    }
  }
}
```

We can see that default it is hidden because of `memoizedState`, if it becomes null, it means it is visible.

When it is visible, it goes down to its child, and will keep trying until find the first HostComponent or HostText.

Then it sets the Host component to hidden by `hideInstance()` and `unhideInstance()` if visible.
Notice that the while loop only goes down to first level.

```js
export function hideInstance(instance: Instance): void {
  // TODO: Does this work for all element types? What about MathML? Should we
  // pass host context to this method?
  instance = ((instance: any): HTMLElement);
  const style = instance.style;
  if (typeof style.setProperty === "function") {
    style.setProperty("display", "none", "important");
  } else {
    style.display = "none";
  }
}

export function hideTextInstance(textInstance: TextInstance): void {
  textInstance.nodeValue = "";
}

export function unhideInstance(instance: Instance, props: Props): void {
  instance = ((instance: any): HTMLElement);
  const styleProp = props[STYLE];
  const display =
    styleProp !== undefined &&
    styleProp !== null &&
    styleProp.hasOwnProperty("display")
      ? styleProp.display
      : null;
  instance.style.display = dangerousStyleValue("display", display);
}

export function unhideTextInstance(
  textInstance: TextInstance,
  text: string
): void {
  textInstance.nodeValue = text;
}
```

And it is done by CSS `display:none`. OMG, I thought it was something fancier. So style is set after the DOM change, which results in below if we set a debugger.

![](/static/offscreen-visiblity-style.gif)

Wait, when is the DOM connected ?

It lies in `commitReconciliationEffects(finishedWork)`. [source](https://github.com/facebook/react/blob/8e2f9b086e7abc7a92951d264a6a5d048defd914/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L2316).

```js
function commitReconciliationEffects(finishedWork: Fiber) {
  // Placement effects (insertions, reorders) can be scheduled on any fiber
  // type. They needs to happen after the children effects have fired, but
  // before the effects on this fiber have fired.
  const flags = finishedWork.flags;
  if (flags & Placement) {
    try {
      commitPlacement(finishedWork);
    } catch (error) {
      captureCommitPhaseError(finishedWork, finishedWork.return, error);
    }
    // Clear the "placement" from effect tag so that we know that this is
    // inserted, before any life-cycles like componentDidMount gets called.
    // TODO: findDOMNode doesn't rely on this any more but isMounted does
    // and isMounted is deprecated anyway so we should be able to kill this.
    finishedWork.flags &= ~Placement;
  }
  if (flags & Hydrating) {
    finishedWork.flags &= ~Hydrating;
  }
}
```

It will insert the DOM nodes here, and instantly set to hidden. Because of it is synchronous, browser doesn't have the chance to render the flickering state.

# Summary

Offscreen Component works as follows

1. it has a state of `visible` or `hidden`
2. if `hidden`, it defers the reconcile in OffscreenLane by bailout in the first pass
3. if `visible`, it just normally reconcile
4. in `completeWork`
   - `Visibility` flag is set if it changes
   - visible DOM is inserted here
5. in commit phase
   - hidden DOM is insert here.
   - React hides / unhides the DOM nodes if `Visibility` flag exists

You might wonder why makes it so complex? We can just use the CSS trick by styles. Yep the whole purpose of this process is to **put the rendering of hidden stuff into a lower priority** -> this is the gold of concurrent mode.

---
layout: post
title: "How does React.memo() work? - React source code walkthrough 14"
date: 2022-01-10 01:21:16 +0900
categories: React
---

> You can watch my [Youtube video explanation](https://www.youtube.com/watch?v=0jbV6apamhs) for this post

In [How does React bailout work in reconciliation]({% post_url 2022-01-08-how-does-bailout-work %}), we managed to understand how React stops reconciling when it finds there is nothing changed in the subtree.

Wait, sounds just like what `React.memo()` does, right?

## Demo time

Again let's open [the old demo](/demos/react/how-bailout-works/index.html), where C and D are both rerendered when the button is clicked.

Now let's change the code a little bit, applying `React.memo()` to D.

Here is the [new demo with memo](/demos/react/how-bailout-works/memo.html), repeat the process, we can see now D is not rerendered, as we'd expected.

![](/static/memo-1.png)

## React.memo() creates a new element: REACT_MEMO_TYPE

First we set a debugger at where `React.memo()` is used, it leads us to the [source code of memo()](https://github.com/facebook/react/blob/main/packages/react/src/ReactMemo.js)

It is pretty simple.

```js
export function memo<Props>(
  type: React$ElementType,
  compare?: (oldProps: Props, newProps: Props) => boolean
) {
  const elementType = {
    $$typeof: REACT_MEMO_TYPE,
    type,
    compare: compare === undefined ? null : compare,
  };
  return elementType;
}
```

As the code explains itself, `memo()` create an element with

1. `$$typeof` to `REACT_MEMO_TYPE`
2. `type` set to our function passed in which is `D()`
3. a `compare` function.

> TBH, I didn't know that `React.memo()` accepts a second argument..

We now know that `React.memo()` creates a new fiber node that wraps things up. Looks straightforward, since if we are to add extra logic to avoid rerendering in `D()`, then we need some thing to wrap it up with that logic.

## reconciliation of MemoComponent

Now again, let's go to [beginWork()](https://github.com/facebook/react/blob/ceee524a8f45b97c5fa9861aec3f36161495d2e1/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L3649) as we explained in the last post, where fiber nodes are checked and updated.

We can easily find where `REACT_MEMO_TYPE` is used. [source](https://github.com/facebook/react/blob/ceee524a8f45b97c5fa9861aec3f36161495d2e1/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L3827-L3862)

```js
case MemoComponent: {
  const type = workInProgress.type;
  const unresolvedProps = workInProgress.pendingProps;
  // Resolve outer props first, then resolve inner props.
  let resolvedProps = resolveDefaultProps(type, unresolvedProps);
  resolvedProps = resolveDefaultProps(type.type, resolvedProps);
  return updateMemoComponent(
    current,
    workInProgress,
    type,
    resolvedProps,
    renderLanes,
  );
}
case SimpleMemoComponent: {
  return updateSimpleMemoComponent(
    current,
    workInProgress,
    workInProgress.type,
    workInProgress.pendingProps,
    renderLanes,
  );
}
```

There are actually two components, one is `MemoComponent`, and one is `SimpleMemoComponent`, we'll understand why soon.

Hey, why is it `MemoComponent`, not `REACT_MEMO_TYPE`? `REACT_MEMO_TYPE` is `$$typeof` used for the element, `MemoComponent` is the function.

When the fiber is created from element, tag is set to `MemoComponent` [here](https://github.com/facebook/react/blob/a724a3b578dce77d427bef313102a4d0e978d9b4/packages/react-reconciler/src/ReactFiber.old.js#L540-L542).

## updateMemoComponent()

`updateMemoComponent()` is is the core of how `React.memo()` works. [source](https://github.com/facebook/react/blob/f2a59df48bec2352a4dd3b5415282a4ea240a4f8/packages/react-reconciler/src/ReactFiberBeginWork.old.js)

```js
function updateMemoComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes
): null | Fiber {
  if (current === null) {
    const type = Component.type;
    if (
      isSimpleFunctionComponent(type) &&
      Component.compare === null &&
      // SimpleMemoComponent codepath doesn't resolve outer props either.
      Component.defaultProps === undefined
    ) {
      let resolvedType = type;
      // If this is a plain function component without default props,
      // and with only the default shallow comparison, we upgrade it
      // to a SimpleMemoComponent to allow fast path updates.
      workInProgress.tag = SimpleMemoComponent;
      workInProgress.type = resolvedType;

      return updateSimpleMemoComponent(
        current,
        workInProgress,
        resolvedType,
        nextProps,
        renderLanes
      );
    }

    const child = createFiberFromTypeAndProps(
      Component.type,
      null,
      nextProps,
      workInProgress,
      workInProgress.mode,
      renderLanes
    );
    child.ref = workInProgress.ref;
    child.return = workInProgress;
    workInProgress.child = child;
    return child;
  }
  const currentChild = ((current.child: any): Fiber); // This is always exactly one child
  const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
    current,
    renderLanes
  );
  if (!hasScheduledUpdateOrContext) {
    // This will be the props with resolved defaultProps,
    // unlike current.memoizedProps which will be the unresolved ones.
    const prevProps = currentChild.memoizedProps;
    // Default to shallow comparison
    let compare = Component.compare;
    compare = compare !== null ? compare : shallowEqual;
    if (compare(prevProps, nextProps) && current.ref === workInProgress.ref) {
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
  }
  // React DevTools reads this flag.
  workInProgress.flags |= PerformedWork;
  const newChild = createWorkInProgress(currentChild, nextProps);
  newChild.ref = workInProgress.ref;
  newChild.return = workInProgress;
  workInProgress.child = newChild;
  return newChild;
}
```

Despite the details, we can see the basic logic is

```text

if (first time) {
  if (is simple) {
    update fiber tag to SimpleMemoComponent, so next time we go directly to updateSimpleMemoComponent()
    updateSimpleMemoComponent()
  } else {
    create child fibers
  }
} else {
  if (we can bailout) {
    try bailout
  } else {
    reconcile children
  }
}
```

## SimpleMemoComponent is the internal optimization

```js
function updateSimpleMemoComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes
): null | Fiber {
  if (current !== null) {
    const prevProps = current.memoizedProps;
    if (
      shallowEqual(prevProps, nextProps) &&
      current.ref === workInProgress.ref &&
      // Prevent bailout if the implementation changed due to hot reload.
      (__DEV__ ? workInProgress.type === current.type : true)
    ) {
      didReceiveUpdate = false;
      if (!checkScheduledUpdateOrContext(current, renderLanes)) {
        workInProgress.lanes = current.lanes;
        return bailoutOnAlreadyFinishedWork(
          current,
          workInProgress,
          renderLanes
        );
      } else if ((current.flags & ForceUpdateForLegacySuspense) !== NoFlags) {
        // This is a special case that only exists for legacy mode.
        // See https://github.com/facebook/react/pull/19216.
        didReceiveUpdate = true;
      }
    }
  }
  return updateFunctionComponent(
    current,
    workInProgress,
    Component,
    nextProps,
    renderLanes
  );
}
```

`updateSimpleMemoComponent()` is much simpler:

1. it just shallow compares the props and try to bailout
2. it doesn't create new fiber, which means there is no `D()` in the fiber tree.

We can see the fiber tree from console. (watch [my video here](https://youtu.be/0jbV6apamhs?t=758) to see how)

![](/static/memo-2.png)

Yeah, the memo element has type of `D()` but its child is going directly to `div`.

if we set pass down a compare function, then React couldn't treat it as simple any more

```js
const D = React.memo(_D, (a, b) => true);
```

Now if we inspect the fiber tree, things are different.

![](/static/memo-3.png)

1. tag is 14, not simple now
2. type is not `D` any more
3. child is `div` but `D`

Yeah, in short, **SimpleMemoComponent is an internal optimization for function components, which merges the memo fiber and wrapped fiber into one**

It looks similar to [View Flattening in React-native](https://reactnative.dev/docs/view-flattening).

And the original pr for this optimization could be found here [https://github.com/facebook/react/pull/13903](https://github.com/facebook/react/pull/13903).

## checkScheduledUpdateOrContext() is always called

Notice that though in `updateMemoComponent()` and `updateSimpleMemoComponent()`, the props are compared but `checkScheduledUpdateOrContext()` is always run, because the wrapped component could have being scheduled update by other functions, like some events or context.

So there are cases though props are equal but bailout does not happen.

Which means, as [React hompeage](https://reactjs.org/docs/react-api.html#reactmemo) says, `React.memo()` is only for performance optimization, Do not rely on it to “prevent” a render.

That's it for `React.memo()`, hope it helps.

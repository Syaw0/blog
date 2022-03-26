---
layout: post
title: "How does useRef() work? - React source code walkthrough 11"
date: 2021-12-05 18:21:10 +0900
categories: React
image: /static/12-use-ref.jpeg
---

> You can watch my [Youtube video explanation](https://www.youtube.com/watch?v=q-B5XalyNpI) for this post

I bet you have used `useRef()` quite a lot, let's figure out its internals in React.

## 1. intro of useRef()

We create a ref by `useRef()`, then programmatically set its `.current` property, or use it as `ref` prop on DOM elements.

```jsx
function Component() {
  const ref = useRef(null);
  return <div ref={ref} />;
}
```

So let's try to solve 2 questions here.

1. how `useRef()` works in initial render and rerender?
2. how `ref={ref}` works?

## 2. how `useRef()` works?

As covered before, let's search directly to `mountRef()` and `updateRef()` which is used in initial render and rerender, and yeah bingo.

### 2.1. mountRef()

[source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.old.js#L1586)

```js
function mountRef<T>(initialValue: T): {| current: T |} {
  const hook = mountWorkInProgressHook();
  const ref = { current: initialValue };
  hook.memoizedState = ref;
  return ref;
}
```

It is super simple

1. create a ref objec which has `current` property
2. create a new hook by `mountWorkInProgressHook()`
3. set ref to `memoizedState` for the ref.

We can ignore for the naming of `memoizedState`, it is just some internal naming for hooks to hold the state stuff.

`mountRef()` is pretty straightforward, I guess for `updateRef()`, we'd just return the `ref` object again? Let's move on

### 2.2. updateRef()

[source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.old.js#L1658)

```js
function updateRef<T>(initialValue: T): {| current: T |} {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState;
}
```

Yep, so simple.

`updateWorkInProgressHook()` as mentioned in my previous videos, it is just to go through the hook list for the fiber, with a internal cursor. For initial render, the list is empty so it creates a new hook every time. For rerender, there is already a hook there, so it just use that hook.

## 3. How `ref={ref}` works?

`useRef()` is so simple, programmatically setting `current` is just change the value of that property, so different from `useState()` it doesn't trigger update.

More interesting question is how it set with DOM element, which includes 2 sub questions.

1. how `ref` is attached
2. how `ref` is detached

### 3.1 how `ref` is attached

For details about how I figured it out, you can refer to [my video](https://www.youtube.com/watch?v=q-B5XalyNpI), here I just listed up the important parts.

During commit phase, `commitAttachRef()` is called inside of `commitLayoutEffectOnFiber()` ([source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberCommitWork.old.js#L685)).

```js
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    const instance = finishedWork.stateNode;
    let instanceToUse;
    switch (finishedWork.tag) {
      case HostComponent:
        instanceToUse = getPublicInstance(instance);
        break;
      default:
        instanceToUse = instance;
    }
    // Moved outside to ensure DCE works with this flag
    if (enableScopeAPI && finishedWork.tag === ScopeComponent) {
      instanceToUse = instance;
    }
    if (typeof ref === "function") {
      let retVal;
      retVal = ref(instanceToUse);
    } else {
      ref.current = instanceToUse;
    }
  }
}
```

We can see that for DOM element, which is HostComponent, the DOM node is set here. Also it accept callback ref or ref object.

This means that the attaching of ref happens the same phase as layout effect, if callback ref is used here, then the attaching could be sooner than `useEffect`, this could helps us understand [React advanced patterns - Reusable behavior hooks through Ref]({% post_url 2022-02-16-reusable-behavior-hooks-through-ref %}).

### 3.2 how `ref` is detached

Below this function there is the detaching function

```js
function commitDetachRef(current: Fiber) {
  const currentRef = current.ref;
  if (currentRef !== null) {
    if (typeof currentRef === "function") {
      currentRef(null);
    } else {
      currentRef.current = null;
    }
  }
}
```

We can see it also support callback ref or ref object. Search for `commitDetachRef()` we can see that it happens in `commitMutationEffectsOnFiber()`, which is even sooner than `commitLayoutEffectOnFiber()`, which is reasonable, because we need to detach first to keep consistency.

`commitMutationEffectsOnFiber()

### 3.3. React knows if needed to attach or detach by `flags`

There is some checks before above functions are called.

```js
if (finishedWork.flags & Ref) {
  commitAttachRef(finishedWork);
}

if (flags & Ref) {
  const current = finishedWork.alternate;
  if (current !== null) {
    commitDetachRef(current);
  }
}
```

They use the same flag `Ref`, if `Ref` flag is set it means ref has changed, for the detaching, it checks `current`, if it is not null then detach the ref on the older fiber, because the fiber is no longer used.

When is `Ref` set? It is done in `markRef()` ([source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L969))

```js
function markRef(current: Fiber | null, workInProgress: Fiber) {
  const ref = workInProgress.ref;
  if (
    (current === null && ref !== null) ||
    (current !== null && current.ref !== ref)
  ) {
    // Schedule a Ref effect
    workInProgress.flags |= Ref;
    if (enableSuspenseLayoutEffectSemantics) {
      workInProgress.flags |= RefStatic;
    }
  }
}
```

We can see that it checks for ref creation and ref change.

`markRef()` is called in `updateHostComponent()` which is inside of reconciliation. ([source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L1466))

```js
function updateHostComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
) {
  pushHostContext(workInProgress);

  if (current === null) {
    tryToClaimNextHydratableInstance(workInProgress);
  }

  const type = workInProgress.type;
  const nextProps = workInProgress.pendingProps;
  const prevProps = current !== null ? current.memoizedProps : null;

  let nextChildren = nextProps.children;
  const isDirectTextChild = shouldSetTextContent(type, nextProps);

  if (isDirectTextChild) {
    // We special case a direct text child of a host node. This is a common
    // case. We won't handle it as a reified child. We will instead handle
    // this in the host environment that also has access to this prop. That
    // avoids allocating another HostText fiber and traversing it.
    nextChildren = null;
  } else if (prevProps !== null && shouldSetTextContent(type, prevProps)) {
    // If we're switching from a direct text child to a normal child, or to
    // empty, we need to schedule the text content to be reset.
    workInProgress.flags |= ContentReset;
  }

  markRef(current, workInProgress);
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

Cool, we now know what is going on for refs.

## 4. Summary

1. during reconciliation, ref changes/creation will be marked on fiber in `flags`
2. during committing, react will detach/attach the ref by checking `flags`
3. `useRef()` is a simple hook which just holds the ref object.

---
layout: post
title: "How does Context work? - React source code walkthrough 9"
date: 2021-07-28 18:21:10 +0900
categories: React
image: /static/context.png
---

> You can watch my [Youtube video explanation](https://www.youtube.com/watch?v=nygcudhNuII) for this post.

As we all know, React internally holds a tree-like structure of fiber nodes, we can pass down data by `props` but it is cumbersome if the tree is very deep. For example, if we want to define a global color which might be used in all components, using `props` leads to possibly creating a new prop on all components.

[Context](https://reactjs.org/docs/context.html) is to solve this isssue, it allows us to pass down data to sub-tree without `props`, you can think of it like a special channel in another dimension.

## 1. demo

Here is [a simple demo of Context](/demos/react/context/), it simply renders `JSer`.

```js
const Context = React.createContext("123");

function Component1() {
  return <Component2 />;
}

function Component2() {
  return <Context.Consumer>{(value) => value}</Context.Consumer>;
}

function App() {
  return (
    <Context.Provider value="JSer">
      <Component1 />
    </Context.Provider>
  );
}

const rootElement = document.getElementById("container");
ReactDOM.createRoot(rootElement).render(<App />);
```

THe demo is very simple, let's see how it Context works internally.

## 2 React.createContext()

The [source code](https://github.com/facebook/react/blob/main/packages/react/src/ReactContext.js) is simple.

```js
import { REACT_PROVIDER_TYPE, REACT_CONTEXT_TYPE } from "shared/ReactSymbols";

import type { ReactContext } from "shared/ReactTypes";

export function createContext<T>(defaultValue: T): ReactContext<T> {
  // TODO: Second argument used to be an optional `calculateChangedBits`
  // function. Warn to reserve for future use?

  const context: ReactContext<T> = {
    $$typeof: REACT_CONTEXT_TYPE,
    // As a workaround to support multiple concurrent renderers, we categorize
    // some renderers as primary and others as secondary. We only expect
    // there to be two concurrent renderers at most: React Native (primary) and
    // Fabric (secondary); React DOM (primary) and React ART (secondary).
    // Secondary renderers store their context values on separate fields.
    _currentValue: defaultValue,
    _currentValue2: defaultValue,
    // Used to track how many concurrent renderers this context currently
    // supports within in a single renderer. Such as parallel server rendering.
    _threadCount: 0,
    // These are circular
    Provider: (null: any),
    Consumer: (null: any),

    // Add these to use same hidden class in VM as ServerContext
    _defaultValue: (null: any),
    _globalName: (null: any),
  };

  context.Provider = {
    $$typeof: REACT_PROVIDER_TYPE,
    _context: context,
  };

  context.Consumer = context;

  return context;
}
```

So nothing is very fancy here, it is just an object holding the default value and exposing `Provider` and `Consumer`

1. `Provider` is a special element type of `REACT_PROVIDER_TYPE`, which we'll cover it later

2. `Consumer` is interesting, it is set to `context`

Notice that context itself is not in the fiber tree, rather like our demo code above, it is the `Provider` and `Consumer` that is used in the rendering. `createContext()` separately allows us to bind `Provider` and `Consumer` together.

## 2 Provider

Element type `REACT_PROVIDER_TYPE` is mapped to fiber tag `ContextProvider` [here](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiber.new.js#L543-L545).

Let's think what should happen during reconciliation of Provider, well the sole purpose of Provider is to set value to the internal context it holds, here is the code.

```js
function beginWork() {
  case ContextProvider:
      return updateContextProvider(current, workInProgress, renderLanes);
}

function updateContextProvider(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  const providerType: ReactProviderType<any> = workInProgress.type;
  const context: ReactContext<any> = providerType._context;

  const newProps = workInProgress.pendingProps;
  const oldProps = workInProgress.memoizedProps;

  const newValue = newProps.value;

  pushProvider(workInProgress, context, newValue);

  if (enableLazyContextPropagation) {
    // In the lazy propagation implementation, we don't scan for matching
    // consumers until something bails out, because until something bails out
    // we're going to visit those nodes, anyway. The trade-off is that it shifts
    // responsibility to the consumer to track whether something has changed.
  } else {
    if (oldProps !== null) {
      const oldValue = oldProps.value;
      if (is(oldValue, newValue)) {
        // No change. Bailout early if children are the same.
        if (
          oldProps.children === newProps.children &&
          !hasLegacyContextChanged()
        ) {
          return bailoutOnAlreadyFinishedWork(
            current,
            workInProgress,
            renderLanes,
          );
        }
      } else {
        // The context value changed. Search for matching consumers and schedule
        // them to update.
        propagateContextChange(workInProgress, context, renderLanes);
      }
    }
  }

  const newChildren = newProps.children;
  reconcileChildren(current, workInProgress, newChildren, renderLanes);
  return workInProgress.child;
}
```

1. it first update the new value by `pushProvider()`
2. if value doesn't change, try [bailout]({% post_url 2022-01-08-how-does-bailout-work %}). if changed, then `propagateContextChange()`
3. go to its children `reconcileChildren()`.

Pretty straightforward.

### 2.1 pushProvider()

([source](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberNewContext.new.js#L87))

```js
export function pushProvider<T>(
  providerFiber: Fiber,
  context: ReactContext<T>,
  nextValue: T
): void {
  if (isPrimaryRenderer) {
    push(valueCursor, context._currentValue, providerFiber);

    context._currentValue = nextValue;
  } else {
    push(valueCursor, context._currentValue2, providerFiber);

    context._currentValue2 = nextValue;
  }
}
```

So it internally uses this [fiber stack](https://github.com/facebook/react/blob/49f8254d6dec7c6de789116e20ffc5a9401aa725/packages/react-reconciler/src/ReactFiberStack.new.js#L59) structure. We can think **fiber stack holds infomation along the path from root to current fiber**.

We can see that in `pushProvider()`:

1. current value is stored in the fiber stack
2. new value is set to the context

For cases like multiple provider on the same context, this makes sure that for a fiber node, the closest value is set when used.

Pay attention that the fiber stack is storing the **previous value** while context itself is set to the newest value, this is because context has default values.

### 2.2 popProvider()

Where there is push, there is pop, called in [completeWork](https://github.com/facebook/react/blob/49f8254d6dec7c6de789116e20ffc5a9401aa725/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L1259-L1264).

This is to make sure that the context is relecting the right value while the fiber tree is traversed by `workInProgress`. Think about an element as a sibling to a Provider, when React goes to this fiber, it should NOT have any information of its sibling Provider, so need to pop.

```js
export function popProvider(
  context: ReactContext<any>,
  providerFiber: Fiber
): void {
  const currentValue = valueCursor.current;
  pop(valueCursor, providerFiber);
  if (isPrimaryRenderer) {
    if (
      enableServerContext &&
      currentValue === REACT_SERVER_CONTEXT_DEFAULT_VALUE_NOT_LOADED
    ) {
      context._currentValue = context._defaultValue;
    } else {
      context._currentValue = currentValue;
    }
  } else {
    if (
      enableServerContext &&
      currentValue === REACT_SERVER_CONTEXT_DEFAULT_VALUE_NOT_LOADED
    ) {
      context._currentValue2 = context._defaultValue;
    } else {
      context._currentValue2 = currentValue;
    }
  }
}
```

So what `popProvider()` does is simple, since fiber stack stores the previous value, just set it to context and pop. done.

In order to understand how `propagateContextChange()`, we need to first understand how `Consumer` works

## 3. Consumer

As mentioned above, `Consumer` actually is the context itself and `ContextConsumer` fiber tag is used ([source](https://github.com/facebook/react/blob/49f8254d6dec7c6de789116e20ffc5a9401aa725/packages/react-reconciler/src/ReactFiber.new.js#L546-L548)).

```js
function updateContextConsumer(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
) {
  let context: ReactContext<any> = workInProgress.type;
  const newProps = workInProgress.pendingProps;
  const render = newProps.children;
  prepareToReadContext(workInProgress, renderLanes);
  const newValue = readContext(context);
  if (enableSchedulingProfiler) {
    markComponentRenderStarted(workInProgress);
  }
  let newChildren;
  newChildren = render(newValue);
  if (enableSchedulingProfiler) {
    markComponentRenderStopped();
  }

  // React DevTools reads this flag.
  workInProgress.flags |= PerformedWork;
  reconcileChildren(current, workInProgress, newChildren, renderLanes);
  return workInProgress.child;
}
```

Ok it is quite straightforward actually

1. it first `prepareToReadContext()`
2. then it reads the value by `readContext()`
3. since consumer expects render prop of children, so `newChildren = render(newValue)`

### 3.1 prepareToReadContext()

```js
export function prepareToReadContext(
  workInProgress: Fiber,
  renderLanes: Lanes
): void {
  currentlyRenderingFiber = workInProgress;
  lastContextDependency = null;
  lastFullyObservedContext = null;

  const dependencies = workInProgress.dependencies;
  if (dependencies !== null) {
    if (enableLazyContextPropagation) {
      // Reset the work-in-progress list
      dependencies.firstContext = null;
    } else {
      const firstContext = dependencies.firstContext;
      if (firstContext !== null) {
        if (includesSomeLane(dependencies.lanes, renderLanes)) {
          // Context list has a pending update. Mark that this fiber performed work.
          markWorkInProgressReceivedUpdate();
        }
        // Reset the work-in-progress list
        dependencies.firstContext = null;
      }
    }
  }
}
```

Looks like it is reseting `dependencies.firstContext`, `dependencies` could be the key of how Privider schedules update to all consumers.

### 3.2 readContext()

```js
export function readContext<T>(context: ReactContext<T>): T {
  const value = isPrimaryRenderer
    ? context._currentValue
    : context._currentValue2;

  if (lastFullyObservedContext === context) {
    // Nothing to do. We already observe everything in this context.
  } else {
    const contextItem = {
      context: ((context: any): ReactContext<mixed>),
      memoizedValue: value,
      next: null,
    };

    if (lastContextDependency === null) {
      if (currentlyRenderingFiber === null) {
        throw new Error(
          "Context can only be read while React is rendering. " +
            "In classes, you can read it in the render method or getDerivedStateFromProps. " +
            "In function components, you can read it directly in the function body, but not " +
            "inside Hooks like useReducer() or useMemo()."
        );
      }

      // This is the first dependency for this component. Create a new list.
      lastContextDependency = contextItem;
      currentlyRenderingFiber.dependencies = {
        lanes: NoLanes,
        firstContext: contextItem,
      };
      if (enableLazyContextPropagation) {
        currentlyRenderingFiber.flags |= NeedsPropagation;
      }
    } else {
      // Append a new context item.
      lastContextDependency = lastContextDependency.next = contextItem;
    }
  }
  return value;
}
```

For a fiber it could use multiple context, that's why the dependencies is actually a linked list.

`readContext()` simply reads the value from the context and update the dependencies.

## 4 propagateContextChange()

When value for a context changes, we need to schedule updates on all the consumers, this is to make sure the update is not skipped somehow.

**Scheduling updates** basically means update `lanes` and `childLanes` for all the node on the path from root to fiber node. You can get more info on this from [How does React bailout work in reconciliation]({% post_url 2022-01-08-how-does-bailout-work %}).

The [source code](https://github.com/facebook/react/blob/49f8254d6dec7c6de789116e20ffc5a9401aa725/packages/react-reconciler/src/ReactFiberNewContext.new.js#L219) has quite some lines, I will not paste it here.

So the idea is quite simple, if you look at the code, you can easily see that the subtree under this Provider is traversed, for each fiber, `dependencies` is checked, if found that the context is used here. `scheduleContextWorkOnParentPath()` is called to schedule some work.

You might ask, wait it is scanned at the `beginWork()` phase of a Provider, what if the consumers are not in the tree yet ?

Good question, at first glance here, the order is a bit strange, since dependencies are only updated after `Consumer` is reconciled. But actually we only need to scan once, because if Consumers nodes are added during the reconciliation, they will automatically use the latest value.

## 5. Summary

The Context is actually pretty straightforward to understand. The core idea is the **fiber stack that allows us to store the information along the path**.

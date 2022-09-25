---
layout: post
title: "How does React portal work internally ? | React Source Code Walkthrough 26"
date: 2022-09-24 18:21:10 +0900
categories: React
image: /static/portal.png
---

- [Demo for React Portal](#demo-for-react-portal)
- [Seems that we can create Portal by ourself?](#seems-that-we-can-create-portal-by-ourself)
- [How does Portal actually work interally?](#how-does-portal-actually-work-interally)
  - [1. createPortal returns special element.](#1-createportal-returns-special-element)
  - [2. createChild() handles Portal differently](#2-createchild-handles-portal-differently)
  - [3. commitPlacement() is where the magic lies.](#3-commitplacement-is-where-the-magic-lies)

## Demo for React Portal

[React Portal](https://reactjs.org/docs/portals.html) is very useful when dealing with modals. It can help put your modal DOM into different layer, while keeping the modal itself where it is in the React Fiber tree and keeping the event propagation.

Let's jump to a [quick demo](/demos/react/portal/).

![](/static//portal-1.gif).

The Modal is rendered inside of App,but the DOM itself is outside of root container.

```jsx
function App() {
  const [showModal, setShowModal] = useState(false);
  return (
    <div>
      <button onClick={() => setShowModal(true)}>show modal</button>
      {showModal && (
        <Modal>
          <div>
            <p>Hello Modal</p>
            <button onClick={() => setShowModal(false)}>hide modal</button>
          </div>
        </Modal>
      )}
    </div>
  );
}
```

![](/static/portal-2.png)

## Seems that we can create Portal by ourself?

Yeah, basically rendering the elements somewhere else right ? Let's try something like below.

```jsx
function Portal({ children, container }) {
  // render the children in a differnet dom
  useLayoutEffect(() => {
    const root = ReactDOM.createRoot(container);
    root.render(children);
    return () => {
      root.unmount(container);
    };
  }, [children]);

  return null;
}

function Modal({ children }) {
  const el = document.createElement("div");
  ...
  return (
    <Portal container={el}>
      <div className="modal-inner">{children}</div>
    </Portal>
  );
}
```

It'll work, in [this demo](/demos/react/portal/by-ourself.html).

Problem is that this approach doesn't allow the modal to inherit any context info, since it is rendered as new React root. Also it creates new React root, which has performance issues.

For example, with the built-in Portal, we can get the context as supposed to, here is [the demo](/demos/react/portal/context.html).

![](/static/portal-3.png)

But with the Portal we built, this doesn't work, here is [the demeo](/demos/react/portal/context-by-ourself.html)

![](/static/portal-4.png)

## How does Portal actually work interally?

Recall in [Initial Mount, how does it work?](https://www.youtube.com/watch?v=EakHciGG3SM), that the syncing from React fiber to real DOM is roughly like this.

1. reconcile -> detect if any fiber changed, if so mark them with flags, like add/removal .etc
   1.1. complete -> create DOM elements for the fibers or reuse them if already exist
2. commit -> for each fiber with those flags, update the DOM accordingly.

One important property on fiber node is `stateNode` which holds the reference to the real DOM (for intrinsic element).

What Portal is special is that only where the DOM is is different, for example if we have structre below.

```
fiber:         Parent > div > Modal > div

DOM(stateNode):     . > div > . > div
```

we can see the stateNode has the real structure in DOM world, but with Portal things should be different.

```
fiber:        Parent > div > Modal > Portal > div

DOM(stateNode):   . > div >   . >

                                             > div
```

What Portal does internally actually is let Portal holds itself a stateNode of the target container element.

```
fiber:        Parent > div > Modal > Portal > div

DOM(stateNode):   . > div >   . >

                                       contain? > div
```

Because of the nature of reconciling, the DOM structure is opaque to React runtime, thus for Portal it could only focus on how to manage the container in the commit phase, everything would work just the same.

### 1. createPortal returns special element.

```js
export function createPortal(
  children: ReactNodeList,
  containerInfo: any,
  implementation: any,
  key: ?string = null
): ReactPortal {
  return {
    // This tag allow us to uniquely identify this as a React Portal
    $$typeof: REACT_PORTAL_TYPE,
    key: key == null ? null : "" + key,
    children,
    containerInfo,
    implementation,
  };
}
```

Nothing special.

### 2. createChild() handles Portal differently

```js
function createChild(
  returnFiber: Fiber,
  newChild: any,
  lanes: Lanes
): Fiber | null {
  if (
    (typeof newChild === "string" && newChild !== "") ||
    typeof newChild === "number"
  ) {
    // Text nodes don't have keys. If the previous node is implicitly keyed
    // we can continue to replace it without aborting even if it is not a text
    // node.
    const created = createFiberFromText("" + newChild, returnFiber.mode, lanes);
    created.return = returnFiber;
    return created;
  }

  if (typeof newChild === "object" && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE: {
        const created = createFiberFromElement(
          newChild,
          returnFiber.mode,
          lanes
        );
        created.ref = coerceRef(returnFiber, null, newChild);
        created.return = returnFiber;
        return created;
      }
      case REACT_PORTAL_TYPE: {
        const created = createFiberFromPortal(
          newChild,
          returnFiber.mode,
          lanes
        );
        created.return = returnFiber;
        return created;
      }
      ...
  }

  return null;
}
```

We can see that `createFiberFromPortal()` is used for portal.

```js
export function createFiberFromPortal(
  portal: ReactPortal,
  mode: TypeOfMode,
  lanes: Lanes
): Fiber {
  const pendingProps = portal.children !== null ? portal.children : [];
  const fiber = createFiber(HostPortal, pendingProps, portal.key, mode);
  fiber.lanes = lanes;
  fiber.stateNode = {
    containerInfo: portal.containerInfo,
    pendingChildren: null, // Used by persistent updates
    implementation: portal.implementation,
  };
  return fiber;
}
```

we can see `stateNode` for Portal is an object holding `containerInfo`, also the fiber type is `HostPortal`.

```js
export function createFiberFromElement(
  element: ReactElement,
  mode: TypeOfMode,
  lanes: Lanes
): Fiber {
  let owner = null;
  const type = element.type;
  const key = element.key;
  const pendingProps = element.props;
  const fiber = createFiberFromTypeAndProps(
    type,
    key,
    pendingProps,
    owner,
    mode,
    lanes
  );
  return fiber;
}
```

Different from `createFiberFromElement()`, where `stateNode` is not set, since we need the DOM to be created inside hierarchy. So it won't be set until commit phase. But for Portal, it already knows where the root is.

### 3. commitPlacement() is where the magic lies.

`commitPlacement()` is what we have mentioned about the real DOM manipulation. `Placement` is the Flag during reconciliation that new DOM needs to be inserted.

```js
function commitPlacement(finishedWork: Fiber): void {
  // Recursively insert all host nodes into the parent.
  const parentFiber = getHostParentFiber(finishedWork);

  // Note: these two variables *must* always be updated together.
  switch (parentFiber.tag) {
    case HostComponent: {
      const parent: Instance = parentFiber.stateNode;
      if (parentFiber.flags & ContentReset) {
        // Reset the text content of the parent before doing any insertions
        resetTextContent(parent);
        // Clear ContentReset from the effect tag
        parentFiber.flags &= ~ContentReset;
      }

      const before = getHostSibling(finishedWork);
      // We only have the top Fiber that was inserted but we need to recurse down its
      // children to find all the terminal nodes.
      insertOrAppendPlacementNode(finishedWork, before, parent);
      break;
    }
    case HostRoot:
    case HostPortal: {
      const parent: Container = parentFiber.stateNode.containerInfo;
      const before = getHostSibling(finishedWork);
      insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
      break;
    }
    // eslint-disable-next-line-no-fallthrough
    default:
      throw new Error(
        "Invalid host parent fiber. This error is likely caused by a bug " +
          "in React. Please file an issue."
      );
  }
}
```

For intrinsic elment - HostComponent, simply append the DOM elements to their parent. The creation of DOM elements are in [completeWork()](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L998).

For Portal - append the DOM elements to the target container. Simple.

That's it, now we know how Portal works internally, actually pretty straightforward thanks to the clean architecture of React runtime.

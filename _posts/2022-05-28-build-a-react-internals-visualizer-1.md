---
layout: post
title: "Build A React Internals Visualizer 1 - PoC"
date: 2022-05-28 18:21:10 +0900
categories: React
image: /static/riv-1.png
---

This is Episode 1 of me trying to build a React Internals Visualizer, you can find the code on [github repo](https://github.com/JSerZANP/react-internals-visualizer) and follow my streaming on [my youtube channel](https://youtube.com/playlist?list=PLvx8w9g4qv_q9NkD-0c9fd7qOBIWiUTw3).

# What is React Internals Visualizer (RIV) ?

If you've watched my previous Youtube videos about React source code walkthrough, you should have seen the diagram I manually made to illustrate how React works, like this one:

![](/static/react-2-memo.png)

This is great, sorta, except that this is not scalable and it is static.

I want to build a visualizer that **automatically illustrate how React works internally against arbitrary React code**

Isn't it super useful to have something like this?

# What would it consist of?

### 1. Inspector

In order to know what is going on inside of React, we need a way to detect the changes of data structure we want to inspect, Inspector is the code we add to React and React-DOM library which send messages to Renderer when needed.

### 2. Message Protocol

Message Protocol is the format being used to communicate between Inspector and Anylizer.

### 3. Analyzer

The messages sent from Inspector need to be analyzed accumulatively to construct a presentation of data structures of React internals.

### 4. Renderer

The final step is to render the internals with an easy-to-understand diagram to users.

# Let's try to build some PoC demo.

If we focus on fiber tree construction and reconciliation, what React does is basically create fiber nodes, connect them into a tree and updates the tree.

We need to detect 2 things

1. when a fiber node is created
2. when a fiber node is updated.

Thus we can defined the message protocol like

```ts
type Fiber = {
  type: "fiber";
  uid: number;
};

type Primitive = null | undefined | string | number;
type Value = Fiber | Primitive;

type MessageCreateFiber = {
  type: "create-fiber";
  payload: {
    fiber: Fiber;
  };
};

type MessageUpdateFiber = {
  type: "update-fiber";
  payload: {
    fiber: Fiber;
    property: string;
    value: Value;
  };
};
```

In order for us to be able to reconstruct the tree structure after receiving the messages,
we need to give the fiber node a unique id., which works like the identifier.

Ok, there is `createFiber()` in react-dom which creates fiber per its name, we returna proxy which enables us to send a message when properties on fiber is updated.

```diff
var createFiber = function (tag, pendingProps, key, mode) {
+  const fiberNode = new FiberNode(tag, pendingProps, key, mode);
+
+  ReactVisualizerInspector.attachUID(fiberNode);
+
+  ReactVisualizerInspector.log({
+    type: "create-fiber",
+    payload: {
+      fiber: fiberNode,
+    },
+  });
+
+  const proxy = new Proxy(fiberNode, {
+    set(target, p, value) {
+      if (ReactVisualizerInspector.meaningfulFiberProperties.includes(p)) {
+        ReactVisualizerInspector.log({
+          type: "update-fiber",
+          payload: {
+            fiber: target,
+            property: p,
+            value,
+          },
+        });
+      }
+      return Reflect.set(target, p, value);
+    },
+  });
-  return new FiberNode(tag, pendingProps, key, mode);
+  return proxy;
};

```

`ReactVisualizerInspector.attachUID` is to assign a unique id to fiber and `log()` will replace fiber with `{UID: string}` to serialize.

```js
const ReactVisualizerInspector = {
  meaningfulFiberProperties: [
    "tag",
    "elementType",
    "type",
    "child",
    "sibling",
    "return",
  ],
  UID: Symbol(),
  isPrimitive: (data) => {
    return (
      data == null ||
      typeof data === "string" ||
      typeof data === "number" ||
      typeof data === "boolean" ||
      typeof data === "symbol" // but we cannot serialize symbol
    );
  },
  uid: (() => {
    let i = 0;
    return () => i++;
  })(),

  attachUID(node) {
    node[ReactVisualizerInspector.UID] = ReactVisualizerInspector.uid();
  },
  normalizeFiber(data) {
    if (ReactVisualizerInspector.isPrimitive(data)) return data;

    if (ReactVisualizerInspector.UID in data) {
      return {
        UID: data[ReactVisualizerInspector.UID],
      };
    }

    return Object.keys(data).reduce((result, key) => {
      result[key] = ReactVisualizerInspector.normalizeFiber(data[key]);
      return result;
    }, {});
  },

  log(message) {
    // traverse through the object and replace fiber which has UID
    const iframe = document.querySelector("#visualizer");
    iframe.contentWindow.postMessage(
      JSON.stringify(ReactVisualizerInspector.normalizeFiber(message)),
      "*"
    );
  },
};
```

Now in the iframe of RIV(React Internals Visualizer), the message is received and parsed to be processed.

```js
function App() {
  const [shapes, setShapes] = React.useState([]);
  React.useEffect(() => {
    window.addEventListener("message", (e) => {
      const messsage = JSON.parse(e.data);
      process(messsage);
      setShapes(toShapes());
    });

    window.parent.postMessage("visualizer-ready", "*");
  }, []);

  return <Renderer shapes={shapes} />;
}

const root = ReactDOM.createRoot(document.getElementById("container"));
root.render(<App />);
```

Ok, the code is shitty here. But it simply does this

1. `process()` will process the message and construct the data structure accumulatively.
2. `toShapes()` will construct the data structure into diagram shapes
3. `<Renderer>` renders the shapes in svg.

The algorithm to construct the tree is very rough and only works for PoC purpose.

```js
const fiberTreeRoots = new Set();
const fiberMap = new Map();

const process = (message) => {
  switch (message.type) {
    case "create-fiber":
      // 1. create a new Node in fiberMap Map<UID, node>
      // 2. put the node in fiberTree
      const UID = message.payload.fiber.UID;
      const node = {
        UID: message.payload.fiber.UID,
      };

      fiberMap.set(UID, node);
      fiberTreeRoots.add(node);
      break;
    case "delete-fiber":
      console.log("delete fiber");
      break;
    case "update-fiber":
      // 1. if `child` or `sibling` is set , reflect udpates on the fiber, and remove it from fiberTree root
      // 2. if `child` or `sibling` is set to null, put the previous child or sibilng onto the fiberTree root
      const property = message.payload.property;
      switch (property) {
        case "child": {
          const childUID = message.payload.value?.UID;
          const child = fiberMap.get(childUID);
          const parent = fiberMap.get(message.payload.fiber.UID);

          if (child == null) {
            const prevChild = parent.child;
            if (prevChild != null) {
              // restore it as root
              fiberTreeRoots.add(prevChild);
            }
          } else {
            fiberTreeRoots.delete(child);
          }
          parent.child = child;
          break;
        }
        case "sibling": {
          const siblingUID = message.payload.value?.UID;
          const sibling = fiberMap.get(siblingUID);
          const from = fiberMap.get(message.payload.fiber.UID);

          if (sibling == null) {
            const prevSibling = from.sibling;
            if (prevSibling != null) {
              // restore it as root
              fiberTreeRoots.add(prevSibling);
            }
          } else {
            fiberTreeRoots.delete(sibling);
          }

          from.sibling = sibling;
          break;
        }
        case "elementType":
        case "type":
        case "tag":
          const target = fiberMap.get(message.payload.fiber.UID);
          target[property] = message.payload.value;
          break;
        default:
          // TODO
          break;
      }
      break;
    default:
      throw new Error("unrecognized message type", message);
  }
};
```

And you can ignore the details above, we'll ditch the code anyway.

`toShapes()` is easy to understand it recursively create the shapes.

```js
const toShapes = () => {
  const shapes = [];

  const walk = (fiber, pos) => {
    shapes.push({
      type: "rectangle",
      label: fiber.elementType || fiber.type, // TODO figure it out what these are
      width: 100,
      height: 40,
      pos,
      key: fiber.UID,
    });

    if (fiber.child) {
      walk(fiber.child, { x: pos.x, y: pos.y + 50 });
    }

    if (fiber.sibling) {
      walk(fiber.sibling, { x: pos.x + 150, y: pos.y });
    }
  };

  const roots = [...fiberTreeRoots];
  for (let i = 0; i < roots.length; i++) {
    walk(roots[i], { x: 150 * i, y: 50 });
  }

  return shapes;
};
```

And Renderer puts shapes into svg canvas.

```js
function renderShape(shape) {
  // TODO: handle more shapes
  switch (shape.type) {
    case "rectangle":
      return [
        <rect
          width={shape.width}
          height={shape.height}
          x={shape.pos.x}
          y={shape.pos.y}
          fill="transparent"
          stroke="#000"
        />,
        <text
          x={shape.pos.x + shape.width / 2}
          y={shape.pos.y + shape.height / 2}
          text-anchor="middle"
          fill="#000"
        >
          {shape.label}
        </text>,
      ];
    default:
      throw new Error("unrecognized shape", shape);
  }
}
function Renderer({ shapes }) {
  return (
    <svg
      version="1.1"
      width="1000"
      height="500"
      xmlns="http://www.w3.org/2000/svg"
    >
      {shapes.map(renderShape)}
    </svg>
  );
}
```

The whole commit could be found [here](https://github.com/JSerZANP/react-internals-visualizer/commit/5f5533ead72de238988f19c36512cb37402d423e#diff-dfb07ec1b2270ab54cda71ee7b25d36428cf7300ddd1a4d4498c8bf812160e86R33031-R33034), And now we can get something like below.

![](/static/riv-ep1-outcome.jpeg).

The diagram is shitty I agree, but we can already see the somewhat tree structure in the first sub-tree to the left, that is exactly what we want.

# It could potentially work. Right?

We've managed to create a PoC demo, it requires hell a lot effort to build something nicer, to even include something about hooks, but it could work I think, right?

Yeah, stay tuned for my next episode, we can bring more stuff into RIV. In the meantime, if you have great idea, you can PR to [github repo](https://github.com/JSerZANP/react-internals-visualizer) as well.

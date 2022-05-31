---
layout: post
title: "Build A React Internals Visualizer 2 - Animated Diagram"
date: 2022-06-01 01:01:10 +0900
categories: React
image: /static/riv-2.png
---

This is Episode 2 of me trying to build a React Internals Visualizer, you can find the code on [github repo](https://github.com/JSerZANP/react-internals-visualizer) and follow my streaming on [my youtube channel](https://youtube.com/playlist?list=PLvx8w9g4qv_q9NkD-0c9fd7qOBIWiUTw3).

# Why animated diagram ?

In [previous episode]({% post_url 2022-05-28-build-a-react-internals-visualizer-1 %}), we've managed to build some static diagram, like below:

![](/static/riv-ep1-outcome.jpeg).

This doesn't help much because it is static, we cannot see how React works internally unless we be able to see how the fiber tree is created and modified.

# Implementation is fairly simple

Instead of flushing the latest snapshot, we can hold all the snapshots and use a cursor to move if forward, with the control of speed.

```diff
+ const IntervalRenderingSnapshot = 500;
function App() {
-  const [shapes, setShapes] = React.useState([]);
+  const [snapshots, setSnapshots] = React.useState([]);
+  const snapshotsRef = React.useRef(snapshots);
+  React.useLayoutEffect(() => {
+    snapshotsRef.current = snapshots;
+  }, [snapshots]);
+
+  const [currentSnapshotIndex, setCurrentShapshotIndex] =
+    React.useState(0);
+
+  // 1. hold all the snapshots
+  // 2. keep track of current cursor on the snapshot list
+  // 3. move forward the cursor with internal
  React.useEffect(() => {
    window.addEventListener("message", (e) => {
      const messsage = JSON.parse(e.data);
      process(messsage);
-      setShapes(toShapes());
+
+      // append the latest snapshot
+      const snapshot = toShapes();
+      setSnapshots((snapshots) => [...snapshots, snapshot]);
    });

    window.parent.postMessage("visualizer-ready", "*");
+
+    window.setInterval(() => {
+      setCurrentShapshotIndex((index) => {
+        const snapshots = snapshotsRef.current;
+        if (snapshots) {
+          return Math.min(snapshots.length - 1, index + 1);
+        }
+        return index;
+      });
+    }, IntervalRenderingSnapshot);
  }, []);

-  return <Renderer shapes={shapes} />;
+  return <Renderer shapes={snapshots[currentSnapshotIndex]} />;
}
```

You can try it out on [the github pages here](https://jserzanp.github.io/react-internals-visualizer/ep2/demo/demo.html).

Besides the above changes, I also fixed the double rendering issue by building the production code without minification, the root fiber stuff.

The whole change could be [found here](https://github.com/JSerZANP/react-internals-visualizer/compare/d3c06e728b45d957d9f0098820033fa23921f9e5...89720ae425bd4912f7ba4234704fa9c097815615)

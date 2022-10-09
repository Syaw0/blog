---
layout: post
title: "Maze Solver Visualizer"
date: 2022-10-09 18:21:10 +0900
categories: React
image: /static/maze-solver.png
---

This is a common algorithm problem which I have never tried it out thouroughly, here I spent 2 hours and built a visualizer, [try it out here](https://stackblitz.com/edit/react-ts-k4ybnb?file=App.tsx,style.css,index.html).

![](/static/maze-1.gif)

The code is [on stackblitz](https://stackblitz.com/edit/react-ts-k4ybnb?file=App.tsx,style.css,index.html) and here just want to summarize my understanding.

## To check if any path exist ? DFS

We can just keep walking around the grid and see if the target cell is stepped upon.

For each move, we have 3 directions, and maximum m\*n cells on the path, thus a tree of O(3^mn) branches is generated.

Well we could have a smaller upper bound, but generally this means time complexity is exponential.

For space, the recursion takes up O(mn).

```js
const checkHasPath = useCallback(() => {
  setIndex(0);
  setResult("");
  const maze = createEmptyMaze(rows, cols, barriers);

  snapshotsRef.current.splice(0);
  function createSnapshot() {
    snapshotsRef.current.push(window.structuredClone(maze));
  }

  createSnapshot();

  function walk(row, col) {
    if (row === rows - 1 && col === cols - 1) {
      setCell(maze, row, col, CellValues.Tried);
      createSnapshot();
      return true;
    }

    if (
      row < 0 ||
      col < 0 ||
      row > rows - 1 ||
      col > cols - 1 ||
      [CellValues.Barrier, CellValues.Tried].includes(getCell(maze, row, col))
    ) {
      return false;
    }

    setCell(maze, row, col, CellValues.Tried);
    createSnapshot();

    // recursively walk on the adjascent cells
    if (
      walk(row + 1, col) ||
      walk(row - 1, col) ||
      walk(row, col + 1) ||
      walk(row, col - 1)
    ) {
      return true;
    }

    setCell(maze, row, col, CellValues.Empty);
    createSnapshot();

    return false;
  }

  const result = walk(0, 0);

  if (result) {
    setResult("found a path with " + snapshotsRef.current.length + " tries");
  } else {
    setResult("path not found");
  }
}, [rows, cols, barriers]);
```

# Improved DFS

The tree has a lot of duplicate trees, if you try out the first option on [our demo](https://stackblitz.com/edit/react-ts-k4ybnb?file=App.tsx,style.css,index.html), sometimes your browser will be frozen, because some failed sub branches are traversed over and over again.

To improve this, we can cache the result of each cell. Meaning if we already know if we can go to the target cell from a cell, we don't need to traverse the subtree.

So now though we have the same tree, every time we walks out of a cell, we remove a cell from the traversal, thus we only need to check 4 cells to determine the result of a cell.

So the time complexity is improved to O(mn) with space extra linear space complexity O(mn)

```js
const checkHasPathImproved = useCallback(() => {
  setIndex(0);
  setResult("");
  const maze = createEmptyMaze(rows, cols, barriers);

  snapshotsRef.current.splice(0);
  function createSnapshot() {
    snapshotsRef.current.push(window.structuredClone(maze));
  }

  createSnapshot();

  const memorized = new Array(rows)
    .fill(0)
    .map((_) => new Array(cols).fill(null));

  function walk(row, col) {
    if (row === rows - 1 && col === cols - 1) {
      setCell(maze, row, col, CellValues.Tried);
      createSnapshot();

      memorized[row][col] = true;
      return true;
    }

    if (
      row < 0 ||
      col < 0 ||
      row > rows - 1 ||
      col > cols - 1 ||
      [CellValues.Barrier, CellValues.Tried].includes(getCell(maze, row, col))
    ) {
      return false;
    }

    const walked = memorized[row][col];
    if (walked != null) {
      return walked;
    }

    setCell(maze, row, col, CellValues.Tried);
    createSnapshot();

    // recursively walk on the adjascent cells
    if (
      walk(row + 1, col) ||
      walk(row - 1, col) ||
      walk(row, col + 1) ||
      walk(row, col - 1)
    ) {
      memorized[row][col] = true;
      return true;
    }

    setCell(maze, row, col, CellValues.Empty);
    createSnapshot();

    memorized[row][col] = false;
    return false;
  }

  const result = walk(0, 0);

  if (result) {
    setResult("found a path within " + snapshotsRef.current.length + " tries");
  } else {
    setResult("path not found");
  }
}, [rows, cols, barriers]);
```

# return any path

This is fairly simple, we just need to keep track of the traversed path in an array, basically the same as above

```js
const findAnyPath = useCallback(() => {
  setIndex(0);
  setResult("");
  const maze = createEmptyMaze(rows, cols, barriers);

  snapshotsRef.current.splice(0);
  function createSnapshot() {
    snapshotsRef.current.push(window.structuredClone(maze));
  }

  createSnapshot();

  const path = [];

  const memorized = new Array(rows)
    .fill(0)
    .map((_) => new Array(cols).fill(null));

  function walk(row, col) {
    if (row === rows - 1 && col === cols - 1) {
      setCell(maze, row, col, CellValues.Tried);
      createSnapshot();

      memorized[row][col] = true;
      return true;
    }

    if (
      row < 0 ||
      col < 0 ||
      row > rows - 1 ||
      col > cols - 1 ||
      [CellValues.Barrier, CellValues.Tried].includes(getCell(maze, row, col))
    ) {
      return false;
    }

    const walked = memorized[row][col];
    if (walked != null) {
      return walked;
    }

    setCell(maze, row, col, CellValues.Tried);
    path.push([row, col]);
    createSnapshot();

    // recursively walk on the adjascent cells
    if (
      walk(row + 1, col) ||
      walk(row - 1, col) ||
      walk(row, col + 1) ||
      walk(row, col - 1)
    ) {
      memorized[row][col] = true;
      return true;
    }

    setCell(maze, row, col, CellValues.Empty);
    path.pop();
    createSnapshot();

    memorized[row][col] = false;
    return false;
  }

  const result = walk(0, 0);

  if (result) {
    setResult(
      "found a path within " +
        snapshotsRef.current.length +
        " tries\n" +
        path.map(([row, col]) => `[${row},${col}]`)
    );
  } else {
    setResult("path not found");
  }
}, [rows, cols, barriers]);
```

# return the shortest path (BFS)

In order to return the shortest path, we don't want to traverse all the possible paths and compare their length, rather we can use BFS to see if we can reach the target cell with 1 steps, 2 steps , 3 steps ...

BFS part is fairly easy, just use an queue and keep dequeueing and enqueueing.

It is a bit tricky to get the path, we need to keep track of the previous cell where the cell is traversed, in our code, we used some special const numbers to mark the direction. Once traversal is done, we go backwards and find the path to the start cell.

```js
const findShortestPath = useCallback(() => {
  setIndex(0);
  setResult("");
  const maze = createEmptyMaze(rows, cols, barriers);

  snapshotsRef.current.splice(0);
  function createSnapshot() {
    snapshotsRef.current.push(window.structuredClone(maze));
  }

  createSnapshot();

  const queue = [[0, 0]];
  setCell(maze, 0, 0, CellValues.Tried);
  createSnapshot();

  let pathFound = false;

  outer: while (queue.length > 0) {
    let length = queue.length;
    while (length > 0) {
      const head = queue.shift();

      if (head[0] === rows - 1 && head[1] === cols - 1) {
        pathFound = true;
        break outer;
      }

      // put adjascent cells in the queue
      const directions = Object.values(Direction);
      for (const direction of directions) {
        const next = getNextCell(maze, head, direction, rows, cols);
        if (next != null) {
          queue.push(next);
          setCell(maze, next[0], next[1], getCellValueFromDirection(direction));
          createSnapshot();
        }
      }

      length -= 1;
    }
  }

  // go backwards to get the shortest path
  if (pathFound) {
    let path = [];
    let cell = [rows - 1, cols - 1];
    while (isCellValueArrow(getCell(maze, cell[0], cell[1]))) {
      const direction = getDirectionFromCellValue(
        getCell(maze, cell[0], cell[1])
      );
      cell = [cell[0] + direction[0], cell[1] + direction[1]];
      path.push(cell);
    }
    path = path.reverse();
    setResult(
      "found a path within " +
        snapshotsRef.current.length +
        " tries\n" +
        " arrows means the incoming cell\n" +
        path.map(([row, col]) => `[${row},${col}]`)
    );
  } else {
    setResult("path not found");
  }
}, [rows, cols, barriers]);
```

Phew, it is a long time since last time I solve algorithm problems. I'm really not good at it.

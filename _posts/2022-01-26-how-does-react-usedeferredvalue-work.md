---
layout: post
title: "How does React.useDeferredValue() work? - React source code walkthrough 17"
date: 2022-01-26 18:21:10 +0900
categories: React
---

> You can watch my [Youtube video explanation](https://www.youtube.com/watch?v=-0-pCZvvwaM) for this post

In React concurrent mode, besides `useTransition()` and `Suspense`, there is one more API called `useDeferredValue()`, let's see what it is and how it works.

# What problem does `useDeferredValue()` try to solve?

There is [detailed answer on React homepage](https://reactjs.org/docs/concurrent-mode-reference.html#usedeferredvalue), but the demo there is broken.

So I'll just quickly demonstrate it here.

## without useDeferredValue()

Open this [demo without useDeferredValue()](/demos/react/usedeferredvalue/no-defer.html).

When the button is clicked, two API mocks for title and posts are fired, title API is quicker (300ms) but the posts API is slow(1000ms). Click the Next button, you can see both title and posts are switched after about 1000ms

![](/static/usedeferredvalue-1.gif)

This is because the button click handler is using `useTransition()`, since error will be thrown until the posts API returns data, both title and posts will be delayed.

This is not good, why don't show title quicker?

## with useDeferredValue()

Now open this [demo with useDeferredValue()](/demos/react/usedeferredvalue/no-defer.html).

Click the Next button again, you can see title is displayed first, followed by posts.

![](/static/usedeferredvalue-2.gif)

This is much better.

# Write useDeferredValue() by ourself

We've already [covered how useTransition() work](https://www.youtube.com/watch?v=G0sHIjjiyJ0&t=2140s), simply speaking it just stops commiting when error is thrown.

To our problem here, when title API returns data React tries to rerender, but when posts is rendered, it throws because data is not ready.

```jsx
function ProfilePage({ resource }) {
  return (
    <React.Suspense fallback={<h1>Loading profile...</h1>}>
      <ProfileDetails resource={resource} />
      <React.Suspense fallback={<h1>Loading posts...</h1>}>
        <ProfileTimeline resource={resource} />
      </React.Suspense>
    </React.Suspense>
  );
}
```

Well, we can cache the `resource` for posts section to solve this problem, by state and update it in `useEffect()`.

```js
function useDeferredValue(value) {
  const [state, setState] = React.useState(value);
  React.useEffect(() => {
    // since value might be promise which causes suspension
    // we should wrap it with startTransition
    React.startTransition(() => {
      setState(value);
    });
  }, [value]);
  return state;
}
```

That's it, now change the code to our implementation.

```jsx
function ProfilePage({ resource }) {
  const deferredResource = useDeferredValue(resource);
  return (
    <React.Suspense fallback={<h1>Loading profile...</h1>}>
      <ProfileDetails resource={resource} />
      <React.Suspense fallback={<h1>Loading posts...</h1>}>
        <ProfileTimeline resource={deferredResource} />
      </React.Suspense>
    </React.Suspense>
  );
}
```

Open [demo with our own useDeferredValue()](/demos/react/usedeferredvalue/my-defer.html), it works just like `React.useDeferredValue()`.

# How does React.useDeferredValue() work?

Set up the debugger, we can find the source code, as usual, one for mount, one for update.

```js
function mountDeferredValue<T>(value: T): T {
  const [prevValue, setValue] = mountState(value);
  mountEffect(() => {
    const prevTransition = ReactCurrentBatchConfig.transition;
    ReactCurrentBatchConfig.transition = 1;
    try {
      setValue(value);
    } finally {
      ReactCurrentBatchConfig.transition = prevTransition;
    }
  }, [value]);
  return prevValue;
}

function updateDeferredValue<T>(value: T): T {
  const [prevValue, setValue] = updateState(value);
  updateEffect(() => {
    const prevTransition = ReactCurrentBatchConfig.transition;
    ReactCurrentBatchConfig.transition = 1;
    try {
      setValue(value);
    } finally {
      ReactCurrentBatchConfig.transition = prevTransition;
    }
  }, [value]);
  return prevValue;
}
```

Wait, it is basically the same as what we wrote!! `mountState` and `updateState` is just implementation for `useState`.

You might wonder what the heck is `ReactCurrentBatchConfig.transition`, let's see the source for `startTransition()` ([source](https://github.com/facebook/react/blob/main/packages/react/src/ReactStartTransition.js))

```js
export function startTransition(scope: () => void) {
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = 1;
  try {
    scope();
  } finally {
    ReactCurrentBatchConfig.transition = prevTransition;
  }
}
```

It is just the same!!! But what on earth is `ReactCurrentBatchConfig.transition` doing?

It is an internal implementation to tell React that updates inside of this function should be scheduled in transition lanes, which leads to NOT commiting when suspended.

The details could be found in [my video](https://www.youtube.com/watch?v=G0sHIjjiyJ0&t=2140s).

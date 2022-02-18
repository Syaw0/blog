---
layout: post
title: "React advanced patterns - Reusable behavior hooks through Ref"
date: 2022-02-18 18:21:10 +0900
categories: React
image: /static/logo.png
---

> This is a blog post for my youtube video [https://www.youtube.com/watch?v=XWj6qbX244Y](https://www.youtube.com/watch?v=XWj6qbX244Y)

## Let's enable passive scroll listener in React

How would you do it? Since React doesn't support by default, you might think about using `useEffect()` and attach the raw listeners, right?

Something like below

```jsx
function PassiveScrollContainer({ onScroll, children }) {
  const refScroll = React.useRef();
  useEffect(() => {
    const scrollContainer = refScroll.current;
    if (scrollContainer) {
      scrollContainer.addEventListener("scroll", onScroll, { passive: true });
      return () => {
        scrollContainer.removeEventListener("scroll", onScroll);
      };
    }
  }, []);
  return <div ref={refScroll}>{children}</div>;
}
```

Above component can be used like this

```jsx
function App() {
  return (
    <PassiveScrollContainer
      onScroll={() => {
        // do something
      }}
    >
      some content
    </PassiveScrollContainer>
  );
}
```

So far so good. But what if we want to customize the styling?

Alright, we can just add `style` to the props

```jsx
function PassiveScrollContainer({onScroll, children, style}) {
   ...
}
```

Sounds good. What if we want to add `onClick` handler to it? what if we need `onMouseEnter` as well?

We can see the problem here, the options will bloat.

**Component level abstraction is not very flexible**. Especially when we are trying to create common components.

## Reuse behaviors hooks through Ref

Notice the jsx in the previous component we created.

```jsx
<div ref={refScroll}>{children}</div>
```

We attached a `ref` to hold the reference to the DOM node, we can actually use its change as a trigger to attach the event listeners.

Let's give it a try

```js
function usePassiveScroll({ onScroll }) {
  const refScroll = useRef();
  useEffect(() => {
    const scrollContainer = refScroll.current;
    if (scrollContainer) {
      scrollContainer.addEventListener("scroll", onScroll, { passive: true });
      return () => {
        scrollContainer.removeEventListener("scroll", onScroll);
      };
    }
  }, [refScroll.current, onScroll]);
}
```

It is used like this.

```js
function App() {
  const onScroll = () => {};
  const refScroll = usePassiveScroll({ onScroll });
  return <div ref={refScroll}>some content</div>;
}
```

This gives us flexibility since now we can use this behavior in any component without intervening the DOM structure.

What if we want multiple behaviors? No worries, let's add a new `useClick()`.

```jsx
function useClick({ onClick }) {
   const refEl = useRef()
   useEffect(() => {
      const el = refEl.current
      if (el) {
         el.addEventListener('click', onClick)
         return () => {
            el.removeEventListener('click', onClick)
         }
      }
   }. [refEl, onClick])
}
```

All we need to do is try to merge the refs together.

This is not difficult, remember that `ref` could be a function?
We can just create a function which sets the current for each ref.

```jsx
const mergeRefs = (...refs) => {
   return useCallback((el) => {
      for (const ref of refs) {
         if ('current' in ref) {
            ref.current = el)
         } else if (type of ref === 'function') {
            ref(el)
         } else {
            throw new Error('not ref')
         }
      }
   }, refs)
}
```

Now both behaviors could be used.

```jsx
function App() {
  const onScroll = () => {};
  const onClick = () => {};
  const refScroll = usePassiveScroll({ onScroll });
  const refClick = useClick({ onClick });
  const mergedRefs = mergeRefs(refScroll, refClick);
  return <div ref={mergedRefs}>some content</div>;
}
```

Neat!

## Wait, we can do it even better with function refs

`useEffect()` on `ref.current` is good, but it is slow, as [I explained in this video](https://www.youtube.com/watch?v=q-B5XalyNpI), function refs is attached at the same phase as `commitLayoutEffects`, sooner than effect hooks.

Let's give it a try.

```jsx
function usePassiveScroll({ onScroll }) {
  const refPrev = useRef();
  const attach = useCallback(
    (el) => {
      el.addEventListener("scroll", onScroll, { passive: true });
      refPrev.current = el;
    },
    [onScroll]
  );
  const dettach = useCallback(
    (e) => {
      refPrev.current.removeEventListener("scroll", onScroll);
      refPrev.current = null;
    },
    [onScroll]
  );

  const ref = (el) => {
    if (el === null) {
      dettach();
    } else {
      attach(e);
    }
  };
  return ref;
}
```

We need to keep track of the `refPrev` because when unmounted `ref()` is called with `null`.

This version with function ref is cleaner and faster.

## useEvent()

Notice how Similar the above 2 hooks are? It is calling us to create a general hook for all events.

```jsx
function useEvent(event, handler, option = {}) {
  const refPrev = useRef();
  const attach = useCallback(
    (el) => {
      el.addEventListener(event, handler, option);
      refPrev.current = el;
    },
    [handler]
  );

  const dettach = useCallback(
    (el) => {
      refPrev.current.removeEventListener(event, handler);
      refPrev.current = null;
    },
    [handler]
  );

  const ref = (el) => {
    if (el === null) {
      dettach();
    } else {
      attach(el);
    }
  };
  return ref;
}
```

Now our code becomes even cooler.

```jsx
function App() {
  const onScroll = () => {};
  const onClick = () => {};
  const refScroll = useEvent('scroll', onScroll,({ onScroll });
  const refClick = useEvent('click', onClick)
  const mergedRefs = mergeRefs(refScroll, refClick);
  return <div ref={mergedRefs}>some content</div>;
}
```

Looks pretty cool, right? How do you think about this pattern?

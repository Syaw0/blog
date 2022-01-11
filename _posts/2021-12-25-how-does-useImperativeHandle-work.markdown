---
layout: post
title: "How does useImperativeHandle() work? - React source code walkthrough 12"
date: 2021-12-25 18:21:10 +0900
categories: React
---

> You can watch my [Youtube video explanation](https://www.youtube.com/watch?v=iuKpQAhunac) for this post

Have you used [useImperativeHandle()](https://reactjs.org/docs/hooks-reference.html#useimperativehandle) before ? Let's figure out how it works intenrally.

## Usage

Here is the [official example usage](https://reactjs.org/docs/hooks-reference.html#useimperativehandle).

```jsx
function FancyInput(props, ref) {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
    },
  }));
  return <input ref={inputRef} />;
}
FancyInput = forwardRef(FancyInput);
```

By above code, we can attatch a ref to `FancyInput` now.

```jsx
function App() {
  const ref = useRef();
  const focus = useCallback(() => {
    ref.current?.focus();
  }, []);
  return (
    <div>
      <FancyInput ref={inputRef} />
      <button onClick={focus} />
    </div>
  );
}
```

Looks straightforward, but why we do this?

## what if we just update ref.current?

Rather than `useImperativeHandle()`, what if we just update `ref.current`? like below.

```js
function FancyInput(props, ref) {
  const inputRef = useRef();
  ref.current = () => ({
    focus: () => {
      inputRef.current.focus();
    },
  });
  return <input ref={inputRef} />;
}
```

It actually works, but there is a problem, `FancyInput` only set `current` of ref accepted, not cleaning up.

Recall our explanation in [React Source Code Walkthrough 11 - how useRef() works?](https://www.youtube.com/watch?v=q-B5XalyNpI), React automatically cleans up refs attached to elements, but now it doesn't.

What if `ref` changes during renders? Then the old ref would still hold the ref which causes problems since from the usage of `<FancyInput ref={inputRef} />`, it should be cleaned.

How to solve this ? We have `useEffect()` which could help clean things up, so we can try things like this.

```jsx
function FancyInput(props, ref) {
  const inputRef = useRef();
  useEffect(() => {
    ref.current = () => ({
      focus: () => {
        inputRef.current.focus();
      },
    });
    return () => {
      ref.current = null;
    };
  }, [ref]);

  return <input ref={inputRef} />;
}
```

but wait, how do you know for sure that `ref` is RefObject not function ref? Ok then, we need to check that.

```js
function FancyInput(props, ref) {
  const inputRef = useRef();
  useEffect(() => {
    if (typeof ref === "function") {
      ref({
        focus: () => {
          inputRef.current.focus();
        },
      });
    } else {
      ref.current = () => ({
        focus: () => {
          inputRef.current.focus();
        },
      });
    }

    return () => {
      if (typeof ref === "function") {
        ref(null);
      } else {
        ref.current = null;
      }
    };
  }, [ref]);

  return <input ref={inputRef} />;
}
```

You know what ? this is actually very similar to how `useImperativeHandle()` works. Except `useImperativeHandle()` is a layout effect, the ref setting happens at same stage of `useLayoutEffect()`, soonner than `useEffect()`.

Btw, Watch my video of explaining useLayoutEffect [https://www.youtube.com/watch?v=6HLvyiYv7HI](https://www.youtube.com/watch?v=6HLvyiYv7HI)

## Ok let's jump into the source code.

For effects, there is mount and update, which are different based on when `useImperativeHandle()` is called.

This is the simplified version of `mountImperativeHandle()`, ([origin code](https://github.com/facebook/react/blob/4729ff6d1f191902897927ff4ecd3d1f390177fa/packages/react-reconciler/src/ReactFiberHooks.new.js#L1799-L1835))

```
function mountImperativeHandle<T>(
  ref: {|current: T | null|} | ((inst: T | null) => mixed) | null | void,
  create: () => T,
  deps: Array<mixed> | void | null,
): void {
  return mountEffectImpl(
    fiberFlags,
    HookLayout,
    imperativeHandleEffect.bind(null, create, ref),
    effectDeps,
  );
}
```

Also for update, [origin code](https://github.com/facebook/react/blob/4729ff6d1f191902897927ff4ecd3d1f390177fa/packages/react-reconciler/src/ReactFiberHooks.new.js#L1837-L1862)

```js
function updateImperativeHandle<T>(
  ref: {| current: T | null |} | ((inst: T | null) => mixed) | null | void,
  create: () => T,
  deps: Array<mixed> | void | null
): void {
  // TODO: If deps are provided, should we skip comparing the ref itself?
  const effectDeps =
    deps !== null && deps !== undefined ? deps.concat([ref]) : null;

  return updateEffectImpl(
    UpdateEffect,
    HookLayout,
    imperativeHandleEffect.bind(null, create, ref),
    effectDeps
  );
}
```

Notice that

1. under the hood `mountEffectImpl` and `updateEffectImpl` are used. `useEffect()` and `useLayoutEffect()` does the same, [here](https://github.com/facebook/react/blob/4729ff6d1f191902897927ff4ecd3d1f390177fa/packages/react-reconciler/src/ReactFiberHooks.new.js#L1698) and [here](https://github.com/facebook/react/blob/4729ff6d1f191902897927ff4ecd3d1f390177fa/packages/react-reconciler/src/ReactFiberHooks.new.js#L1744-L1759)
2. the 2nd argument is `HookLayout`, which means it is layout effect.

Last piece of puzzle, this is how `imperativeHandleEffect()` works. ([code](https://github.com/facebook/react/blob/4729ff6d1f191902897927ff4ecd3d1f390177fa/packages/react-reconciler/src/ReactFiberHooks.new.js#L1769-L1797))

```js
function imperativeHandleEffect<T>(
  create: () => T,
  ref: {| current: T | null |} | ((inst: T | null) => mixed) | null | void
) {
  if (typeof ref === "function") {
    const refCallback = ref;
    const inst = create();
    refCallback(inst);
    return () => {
      refCallback(null);
    };
  } else if (ref !== null && ref !== undefined) {
    const refObject = ref;
    const inst = create();
    refObject.current = inst;
    return () => {
      refObject.current = null;
    };
  }
}
```

Set aside the details of perfectness, it actually looks very much the same as what we wrote, right?

## Wrap-up

`useImperativeHandle()` is no magic, it just wraps the ref setting and cleaning for us, internally it is in the same stage as `useLayoutEffect()` so a bit sooner than `useEffect()`.

---
layout: post
title: "An alternative(?) to React.useEvent() "
date: 2022-05-07 18:21:10 +0900
categories: React
image: /static/useEvent.png
---

Recently there is a hot discussion of [React RFC: useEvent()](https://github.com/reactjs/rfcs/pull/220), if you haven't read it I strongly suggest you spend some time on it.

So `useEvent()` is going to be the recommended way of creating event callbacks which **allows us to easily avoid unnecessary invalidation of useCallback()**.

In this post I'll try to solve the problem naively with existing concepts and see how far we can go.

## 1. What is the problem we are solving?

Here is the basic example to illustrate what the problem is.

```jsx
function _SendButton({ onClick }: { onClick: () => void }) {
  console.log("render SendButton");
  return <button onClick={onClick}>send</button>;
}

const SendButton = React.memo(_SendButton);

export default function App() {
  const [text, setText] = React.useState("");
  const onInput = React.useCallback(
    (e: React.ChangeEvent<HTMLInputElement>) => {
      setText(e.currentTarget.value);
    },
    []
  );

  const send = () => {
    console.log("send:", text);
  };

  return (
    <div>
      <input value={text} onChange={onInput} />
      <SendButton onClick={send} />
    </div>
  );
}
```

When we type something in the input, `SendButton` gets rerendered because `send()` is newly created
on every re-render of App().

In order to solve this problem, we are already familiar with `useCallback()`, we can wrap `send()` with it.

```jsx
const send = React.useCallback(() => {
  console.log("send:", text);
}, []);
```

This stablize `send()` but it introduces a new bug that when button is clicked, `text` inside is always empty.
This is because hooks are like some real hooks that hook extra data to the fiber tree, the data being hooked cannot be updated without explicit dependency change.

So we need to add `text` in the dependency array.

```jsx
const send = React.useCallback(() => {
  console.log("send:", text);
}, [text]);
```

But now this reintroduce the problem we were having, this `send()` is NOT stable, it is recreated every time `text` changes.

## 2. How to solve it with ref?

Ref is the only thing that is persisten across different renders,
with this we can easily solve the problem by wrapping `text` into `textRef`.

```js
const [text, setText] = React.useState("");
const textRef = React.useRef(text);
React.useLayoutEffect(() => {
  textRef.current = text;
}, [text]);
const send = React.useCallback(() => {
  console.log("send:", textRef.current);
}, []);
```

Why `useLayoutEffect()` is used is because setting ref in rendering is not safe in concurrent mode and it is run ealier than `useEffect()`.

Above code is working and works fine, but it becomes ugly if there is multiple `textRef`.

We can put the logic into a separate reusable hook, like below

```js
function useRefData(data) {
  const ref = React.useRef(data);
  React.useLayoutEffect(() => {
    ref.current = data;
  }, [data]);
  return ref;
}

const [text1, setText1] = React.useState("");
const [text2, setText2] = React.useState("");

const textRef1 = useRefData(text1);
const textRef2 = useRefData(text2);

const send = React.useCallback(() => {
  console.log("send:", textRef1.current, textRef2.current);
}, []);
```

This looks better, but still not elegant, what if `useRefData()` supports multiple arguments?

```js
function useRefData(...args) {
  const refs = args.map(React.useRef);
  React.useLayoutEffect(() => {
    for (let i = 0; i < args.length; i++) {
      refs[i].current = args[i];
    }
  }, args);

  return React.useCallback(() => refs.map((ref) => ref.current), []);
}

const [text1, setText1] = React.useState("");
const [text2, setText2] = React.useState("");

const getData = useRefData(text1, text2);

const send = React.useCallback(() => {
  const [text1, text2] = getData();
  console.log("send:", text1, text2);
}, [getData]);
```

Now it looks much better, and extra one dependency `getData()` is more aligned with my understanding of React for now.

## 3. Wait, isn't it the same as `useEvent()`?

Maybe yes, currently `useRefData()` has almost the same conceptual implementation compared to `useEvent()`.
This is actually interesting, because we can think of `useRefData()` is a way to stablize some data through ref,
and we can of course stablize some function with it as well.

```js
const rawCallback = () => {
  console.log("send:", text1, text2);
};

const callback = useRefData(rawCallback)()[0];
```

We can see that we are actually solving the same issue from different angle.

1. `useRefData()`: stablizes the input data
2. `useEvent()`: stablizes the usage

Which one is better ? Not sure but I feel that

1. `useRefData()`: easier to understand but easier to mess up
2. `useEvent()`: needs extra cognitive load but easier to manage.

What is your opinion on this?

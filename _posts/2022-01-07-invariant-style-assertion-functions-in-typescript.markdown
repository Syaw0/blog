---
layout: post
title: "Invariant-style assertion functions in TypeScript"
date: 2022-01-06 18:21:10 +0900
categories: TypeScript
---

There is this [invariant()](https://github.com/facebook/react/blob/v0.13.3/src/vendor/core/invariant.js) in React source code that I found pretty useful.

It is used to assert some condition should be truthy, if not then an error should be thrown, we can simplify it to something like below.

```ts
function invariant(condition: any, message: string) {
  if (!condition) {
    throw new Error(message);
  }
}
```

## cases where it is useful

In a backend API which receives user input from client, we need to do data validation.

```ts
function handleInput(input: string | null) {
  if (input == null) {
    throw new Error("input should not be null");
  }
  // input is string now
  const validaInput = input;
}
```

If we are to do it in variant style, it'll be like this.

```ts
function handleInput(input: string | null) {
  invariant(input, "input should not be null");
  // input is string now
  const validaInput = input;
}
```

But TypeScript doesn't infer that input is not null here,`validaInput` is still `string | null`.

## Assertion Function comes as rescue.

Actually it was added a long time ago in [TypeScript 3.7](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html), we can use assertion function here to tell TypeScript that if invariant doesn't throw an error, then the input is not null.

We need `invariant()` to be a generic function to suit more cases.

```ts
function invariant<T>(
  data: T,
  message: string
): asserts data is NonNullable<T> {
  if (!data) {
    throw new Error(message);
  }
}
```

Now if we [try the code](https://www.typescriptlang.org/play?ssl=5&ssc=2&pln=1&pc=1#code/GYVwdgxgLglg9mABDMA3AhgJxusUA8AKgHwAUAJulOgFyKEA0iAtgKYDO76A5q3e1GxhuASjrpOrTFHaJK1ZLIByCJSAA269ACN1rIsUQBvALAAoRMmCJSAQnnoRx85ctQAFpjgB3RGFa+AKKYXpikbJw8rCIuiAC+5glm5qCQsAiI7rjkegCSYAAOIFCkKEVQ-IIo3IgAPn4a6k6mFshoWDh4pYXFTABEZcWI7O5wGuR+cFCI2qwNmn0iANyxAPSrbeWKw1XCk96xEAgCiBjqMJT5WwC8m8WJQA), `validaInput` has the right type of `string`.

Or we can assert data to be some specific type.

```ts
function assertsString(data: any): asserts data is string {
  if (typeof data !== "string") {
    throw new Error("data should be string");
  }
}
```

Which is basically the same as type guard, but without the usage of `if clause`.

```ts
function isString(data: any): data is string {
  return typeof data === "string";
}

if (isString(foo)) {
  // foo is string
}
```

I personally like invariant-style better, how about you?

> watch my Youtube video for this post: [https://www.youtube.com/watch?v=BGk79_GfT7U](https://www.youtube.com/watch?v=BGk79_GfT7U))

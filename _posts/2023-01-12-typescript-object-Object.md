---
layout: post
title: "`{}` vs `object` vs `Object` in TypeScript."
date: 2023-01-12 11:21:10 +0900
categories: TypeScript
image: /static/ts-obj.png
---

With this post I'll never be confused again about their difference.

## type `Object`

The type `Object` refers to the `Object` constructor in JavaScript.

Recall the prototype chain, (almost) every object has `Object.prototype` at the end of their prototype chain, so (almost) every object is instance of `Object`.

```ts
({}).__proto__ === Object.prototype  // true
([]).__proto__.__proto__) === Object.prototype); // true
new Map().__proto__.__proto__ === Object.prototype; // true
```

For primitive values(except `null` and `undefined`), when calling the methods on them, a wrapper object will be created so primitive values have the prototype chain as well

```ts
const a = 1;
a.__proto__.__proto__ === Object.prototype; // true
const b = symbol();
b.__proto__.__proto__ == Object.prototype; // true
```

So, every non-null values could be typed as `Object`.

```ts
let a: Object = 1;
let b: Object = {};
let c: Object = new Map();
```

But `Object` type is [not recommended officially](https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html#general-types).

> ❌ Don’t ever use the types Number, String, Boolean, Symbol, or Object These types refer to non-primitive boxed objects that are almost never used appropriately in JavaScript code.

## type `{}`

`{}` is an empty object literal and it is (almost) the same as `Object` - meaning they are assignable to each other, TypeScript doesn't complain about code below

```ts
declare let a: {};
declare let b: Object;
a = b;
b = a;
```

Then what's the difference ? The slight different is that Object is more stricter about the prototype methods.

![](/static/ts1.png)

We can see that `Object` strictly checks the return type, but `{}` ignore the return value. I don't know why.

Why I mention `(almost)` in previous section about `Object` is that we actually can create some object which doesn't fall into its category.

```ts
const obj = Object.create(null);
//    ^? const obj: any
```

This `obj` doesn't have prototype, so `obj.toString()` throws an error while TypeScript types it as `any` which is not Sound.

In my humble opinion, `{}` would be a better option to type it if TypeScript has make it explicit about such nuances, since cleary this `obj` is not of type `Object`. But as shown above, `{}` and `Object` doesn't make much difference for now.

## `object`

`object` simply represents non primitive values, meaning not `number`, `string`, `symbol`, `null`, `undefined` or `bigint`. ([ref](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-2.html#object-type))

## Summary

So now we can understand why `unknown` means `{} | null | undefined`, and here is the rule of thumb:

1. for the objects we usually means, use `object`.
2. for non-null values, use `{}`.

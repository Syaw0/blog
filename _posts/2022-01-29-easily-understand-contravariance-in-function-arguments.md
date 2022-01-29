---
layout: post
title: "Easily understand Contravariance of function arguments in TypeScript"
date: 2022-01-29 18:21:10 +0900
categories: TypeScript
image: /static/logo.png
---

> Watch [my video on youtube](https://youtu.be/7YHnsH3KCJE) for this post.

Take a look at this code snippet

```ts
class Animal {}

class Cat extends Animal {
  purr() {}
}

declare let animal: Animal;
declare let cat: Cat;
```

Can `cat` and `animal` be assigned to each other ?

![](/static/contravariance-1.png)

The answer looks simple, `cat` to be assigned to `animal`, but not the other way around. This looks obvious.

Now let's have two functions which accept `Animal` and `Cat`.

```ts
type FeedAnimal = (animal: Animal) => void;
type FeedCat = (cat: Cat) => void;

declare let feedAnimal: FeedAnimal;
declare let feedCat: FeedCat;
```

Can `feedAnimal` and `feedCat` be assigned to each other?

![](/static/contravariance-2.png)

Ok, now `feedCat` cannot be assigned `feedAnimal`, which seems a bit strange, comparing to the previous case.

## Because function argument in TypeScript is Contravariant.

Here is great post explaining [What are covariance and contravariance](https://www.stephanboyer.com/post/132/what-are-covariance-and-contravariance), I won't do it here in my post.

Just I've thought about a great analogy to help easily understand why function argument is Contravariant.

Think about an pipe connector which connects different pipe sizes.

![](/static/contravariance-3.png)

Water flows in and out, the size of entry is bigger and the pipe, the size of exit is smaller than the pipe.

Now if we are to replace a valve, what can we use ?

![](/static/contravariance-4.png)

We definitely need a connector which has even bigger entry and smaller exit, so that it could fit in without water loss.

> Well, in real world, we only choose the size just fits, here is just a analogy.

Think of connector as function, then

1. function argument type is contravariant
2. function return type is covariant

## Should Array&lt;T&gt; be covariant?

TypeScript treats `Array<T>` to be covariant.

![](/static/contravariance-5.png)

This looks straightfoward but it is not sound. Think about following code.

```ts
animalList = catList;
animalList.push(new Dog());

catList.forEach((cat) => cat.purr()); // errors in runtime, but TypeScript doesn't know.
```

TypeScript doesn't complaint, but `animalList` and `catList` points to the same object with different types, which leads to the runtime error.

The topic is deep which myself doesn't know much, but just remember that **TypeScript type system helps but it has flaws in soundness**.

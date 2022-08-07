---
layout: post
title: "Try to build types in TypeScript, not write them"
date: 2022-08-06 18:21:10 +0900
categories: TypeScript
image: /static/build-types.png
---

> You can watch my [Youtube video explanation](https://www.youtube.com/watch?v=AmzfAih2CEk) for this post.

## Managing types in TypeScript is not easy!

Here is an example.

Suppose we have a bunch of classes which extends the base class, which is quite common in real-world applications, we can illustrate the situation like blow.

```ts
class BaseClass {}
class AClass {}
class BClass {}
```

As mentioned in [Inversion of Control](https://www.youtube.com/watch?v=lxePHsJAJ8I), we might have some functions to process these instances, thus there is a need to differentiate the instances to each types. This could lead us to require some unique identifier of each class, let's say it is named as `prop`.

We definitely don't want to type it as `string`, it is too wide, union of string literals would be the perfect case here.

```ts
// props.ts
const props = ["base", "a", "b"] as const;
export type Prop = typeof props[number];

// BaseClass.ts
import BaseClass from "./BaseClass";
import { Prop } from "../types";
class BaseClass {
  prop: Prop = "base";
}

// AClass.ts
import BaseClass from "./BaseClass";
import { Prop } from "../types";
class AClass extends BaseClass {
  prop: Prop = "a";
}

// BClass.ts
import BaseClass from "./BaseClass";
import { Prop } from "../types";
class BClass extends BaseClass {
  prop: Prop = "b";
}
```

Everything looks fine, but there is an issue. **type definition is in separated from where it should be**.

Take a look at `AClass.ts`, the value of `"a"` is used here, but defined in `props.ts`, somewhat creates 2 sources of truth. Now everytime I want to add a new value or modify the existing ones, I need to touch 2 files, which is not scalable. Think about the conflicts we can have in a large organization.

## Let's build the types, not write them.

If we look at the steps we took in above example, we can find that the typing for `Prop` is merely extracting some information from existing code, we are not adding types for some unknow data sources, the manual process could easily be replaced by build scripts.

The same idea could be found in [Relay Compiler](https://relay.dev/docs/guides/compiler/), which removes us from manually hoisting the data fetching.

```ts
// buildTypes.ts

import fs from "fs";

const dir = "./classes";

function readFiles(): Promise<string[]> {
  return new Promise((resolve) => {
    fs.readdir("./classes", (err, files) => {
      resolve(files);
    });
  });
}

async function build() {
  const files = await readFiles();
  const types = await Promise.all(
    files.map((file) => {
      return import(dir + "/" + file).then((module) => {
        return new module.default().prop;
      });
    })
  );
  const uniqueTypes = Array.from(new Set(types));

  if (uniqueTypes.length !== types.length) {
    throw new Error("hey, some classes are using the same prop value");
  }

  // generate built types file
  const typeContent = `
  const props = [${uniqueTypes
    .map((type) => "'" + type + "'")
    .join(",")}] as const;
  export type Prop = typeof props[number];
  `;

  fs.writeFileSync("./built-types.ts", typeContent);
}

build();
```

The above script is pretty straightforward, it fetchs all the classes and generate the TypeScript code, now `types.ts` could just simply re-export the built types.

```ts
// types.ts
export type { Prop } from "./built-types";
```

What this means is that we only need to care about the classes, not the types any more. Anytime we have changes, we just run above script and done. This makes thing easier and more scalable.

Fun part is that there is a uniqueness check in above script, I bet you can do much fancier stuff with it. This shows how powerful scripts can be comparing to the default capabilities from TypeScript.

Also above is just some hacky example, you might want to try AST parsing and accomplish it in a more robust way.

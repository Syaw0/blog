---
layout: post
title: "Definitive guide to React types in TypeScript"
date: 2022-03-27 18:21:10 +0900
categories: React
image: /static/21-lanes.png
---

Let's start small

```ts
const Component: React.FunctionComponent<{ name: string }> = ({ name }) => {
  return <p>{name}</p>;
};
```

```ts
type FC<P = {}> = FunctionComponent<P>;

interface FunctionComponent<P = {}> {
  (props: P, context?: any): ReactElement<any, any> | null;
  propTypes?: WeakValidationMap<P> | undefined;
  contextTypes?: ValidationMap<any> | undefined;
  defaultProps?: Partial<P> | undefined;
  displayName?: string | undefined;
}
```

P means Props, below is a interface for function, see the call signaure.

Notice that the return value is typed as ReactElement<any, any> no matter what.

## ReactElement

```ts
interface ReactElement<
  P = any,
  T extends string | JSXElementConstructor<any> =
    | string
    | JSXElementConstructor<any>
> {
  type: T;
  props: P;
  key: Key | null;
}
```

As we mentioned before, React Element means following structure

```
type
props
key
```

for type, it could be

1. string - means intrinsic html tags
2. Function which could be Class or Function

### JSXElementConstructor

```ts
type JSXElementConstructor<P> =
  | ((props: P) => ReactElement<any, any> | null)
  | (new (props: P) => Component<any, any>);
```

See the new signature.

Now let's type elements with ReactElement

```ts
import * as React from "react";

function Child1({ a }: { a: number }) {
  return null;
}

function Child2() {
  return null;
}
type A1 = React.ReactElement<{ a: 1 }, typeof Child1>;
type A2 = React.ReactElement<typeof Child2>;

let a = React.createElement(Child1, { a: 1 });
let b = <Child1 a={1} />;

type A = React.FunctionComponent;
```

## FunctionComponentElement

```ts
interface FunctionComponentElement<P>
  extends ReactElement<P, FunctionComponent<P>> {
  ref?:
    | ("ref" extends keyof P
        ? P extends { ref?: infer R | undefined }
          ? R
          : never
        : never)
    | undefined;
}
```

it extracts `ref` out.

## JSX.Element

```ts
let a = React.createElement(Child1, { a: 1 });
let b = <Child1 a={1} />;
```

The types are different, the second one is always `JSX.Element`

```ts
declare global {
    namespace JSX {
        interface Element extends React.ReactElement<any, any> { }
```

While `createElement` has more specific types with overload

````ts
function createElement(
        type: "input",
        props?: InputHTMLAttributes<HTMLInputElement> & ClassAttributes<HTMLInputElement> | null,
        ...children: ReactNode[]): DetailedReactHTMLElement<InputHTMLAttributes<HTMLInputElement>, HTMLInputElement>;
    function createElement<P extends HTMLAttributes<T>, T extends HTMLElement>(
        type: keyof ReactHTML,
        props?: ClassAttributes<T> & P | null,
        ...children: ReactNode[]): DetailedReactHTMLElement<P, T>;
    function createElement<P extends SVGAttributes<T>, T extends SVGElement>(
        type: keyof ReactSVG,
        props?: ClassAttributes<T> & P | null,
        ...children: ReactNode[]): ReactSVGElement;
    function createElement<P extends DOMAttributes<T>, T extends Element>(
        type: string,
        props?: ClassAttributes<T> & P | null,
        ...children: ReactNode[]): DOMElement<P, T>;

    // Custom components

    function createElement<P extends {}>(
        type: FunctionComponent<P>,
        props?: Attributes & P | null,
        ...children: ReactNode[]): FunctionComponentElement<P>;
    function createElement<P extends {}>(
        type: ClassType<P, ClassicComponent<P, ComponentState>, ClassicComponentClass<P>>,
        props?: ClassAttributes<ClassicComponent<P, ComponentState>> & P | null,
        ...children: ReactNode[]): CElement<P, ClassicComponent<P, ComponentState>>;
    function createElement<P extends {}, T extends Component<P, ComponentState>, C extends ComponentClass<P>>(
        type: ClassType<P, T, C>,
        props?: ClassAttributes<T> & P | null,
        ...children: ReactNode[]): CElement<P, T>;
    function createElement<P extends {}>(
        type: FunctionComponent<P> | ComponentClass<P> | string,
        props?: Attributes & P | null,
        ...children: ReactNode[]): ReactElement<P>;
        ```
````

````ts
interface ReactNodeArray extends ReadonlyArray<ReactNode> {}
    type ReactFragment = Iterable<ReactNode>;
    type ReactNode = ReactElement | string | number | ReactFragment | ReactPortal | boolean | null | undefined;
    ```
````

Fragment is not Element, as mentioned here TODO

but it is typed as `Iterable<ReactNode>`, don't know why

`nope`, the Fragment here is just a local name

the real type for Fragment is

```ts
const Fragment: React.ExoticComponent<{
  children?: React.ReactNode;
}>;
```

> if you need an array or `Iterable<ReactNode>` if its passed to a host component.

Anyway, all are treated as JSX.Element

Like node and element in DOM, ReactNode is a union of ReactElement | string | .etc.

We can see that ReactNode actually has `Iterable<ReactNode>` which means array is supported.

````ts
    // We don't just use ComponentType or FunctionComponent types because you are not supposed to attach statics to this
    // object, but rather to the original function.
    interface ExoticComponent<P = {}> {
        /**
         * **NOTE**: Exotic components are not callable.
         */
        (props: P): (ReactElement|null);
        readonly $$typeof: symbol;
    }
    ```
````

As mentioned above about Suspense, $$typeof is different.

> **NOTE**: Exotic components are not callable.

why?

```ts
type ReactChild = ReactElement | string | number;
```

```ts
type ComponentProps<
  T extends keyof JSX.IntrinsicElements | JSXElementConstructor<any>
> = T extends JSXElementConstructor<infer P>
  ? P
  : T extends keyof JSX.IntrinsicElements
  ? JSX.IntrinsicElements[T]
  : {};
```

Simply it infers the Props

```ts
type PropsWithChildren<P = unknown> = P & { children?: ReactNode | undefined };
```

Simply adding `children`

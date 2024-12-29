---
layout: post
title:  "TypeScript: immutable data"
date:   2024-11-06 19:00:00 +0100
categories: typescript
---

When working on an Angular app, there's usally some data involved.
If it's some state inside a component or more global in a service, an app without data may not be much useful.

For me there are three main categories of data:

- entered by the user
- received from a backend
- submitted to a backend

Regardless where the source of the data is located, the app has to react to changes of this data.
Currently there are two main reactive primitives: signals and observables.

When we pass around primitive types like `string` or `number` we don't have to worry much.
But most of time we will use more complex objects, because we have to group data together.
It can be the current user, containing a username, a name for display, a list of the claims etc..
Each of these properties in an object can change independent of each other.
But we don't want to mutate a property directly, because then the dependent consumer of the signal or observable doesn't get notified about the change.
They may have just an altered version of the object and don't know it - even worse, everyone with a reference to this object may mutate it.
Sooner or later (sooner of course) this will lead to problems.

It's always a good idea to let shared data be immutable, so everyone agrees on what they have.
And if we want to mutate a property we create a new object with the mutation and pass it through the signal/observable, so all get notified about the change.

With TypeScript we can define properties as `readonly`.
With this in place the TS transpiler will error, when we try to set such a property.

```typescript
type User = {
  readonly username: string;
  readonly displayname: string;
  readonly claims: { type: string; value: string }[];
};
```

But repeating the `readonly` on every property is cumbersome.  
Oh, and we missed the nested type!  
And the array is mutable, too!

```typescript
type User = {
  readonly username: string;
  readonly displayname: string;
  readonly claims: readonly {
    readonly type: string;
    readonly value: string;
  }[];
};
```

That are a lot of `readonly`s...

TypeScript helps us (a bit) with its utility type [`Readonly<Type>`](https://www.typescriptlang.org/docs/handbook/utility-types.html#readonlytype).
With this we can get rid of some of them:

```typescript
type User = Readonly<{
  username: string;
  displayname: string;
  claims: readonly Readonly<{
    type: string;
    value: string;
  }>[];
}>;
```

But we have to declare it on every nested type (if we don't split them) and the `readonly` on the array is something I forget most of the time.

Let's build our own utility type!

Mapping an object to a readonly version of itself is pretty straight forward:

```typescript
export type Immutable<T> = {
  readonly [K in keyof T]: T[K];
}
```

But we don't always have objects, sometime we have primitives or arrays.
So we need some conditions in the right order and add a grain of recursivity.

```typescript
export type Immutable<T> =
  T extends Array<infer E>
    ? ReadonlyArray<Immutable<E>>
    : T extends object
      ? {
          readonly [K in keyof T]: Immutable<T[K]>;
        }
      : T;
```

An array is the easiest thing to detect.
And to keep typesafety we need to infer the type of the elements.
We then just map it to an `ReadonlyArray` and make the elements immutable, too.

Since I don't want to enumerate all primitive types, the next test is, if the type is an object.
We then make every property `readonly` and recurse into the type of each property like we did with the elements of the array.

And as a last resort we just map the given type to itself.
At this point it must be a primitive like `string` or `number`.
And if not - at least we don't break anyone.

And of course you can build the counterpart of this utility type with the same pattern:

```typescript
export type Mutable<T> = T extends Array<infer E> | ReadonlyArray<infer E>
  ? Array<Mutable<E>>
  : T extends object
    ? {
        -readonly [K in keyof T]: Mutable<T[K]>;
      }
    : T;
```

What I totally ignored here are `function`s.
When I transfer data objects between different parts of my app (hence the name "data transfer objects" or "DTOs"), they contain just data and not behavior.

But back to the example:

```typescript
type User = Immutable<{
  username: string;
  displayname: string;
  claims: { type: string; value: string }[];
}>;
```

If you copy this to some TypeScript playground you will see that every (nested) property and also the array will be readonly.

Protect yourself (and your users)!

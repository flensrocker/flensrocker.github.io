---
layout: post
title:  "Angular Preview: httpResource in a nutshell"
date:   2025-02-15 10:00:00 +0200
categories: angular http resource
---

On 2025-02-14 Alex Rickabaugh showed us the new `httpResource` over at [Tech Stack Nation](https://techstacknation.com/).
It will land in Angular v19.2.0.

Beware

- It's experimental.
- Expect changes.
- This is not covered by official documentation, it's what I learned from the presentation and [pull request](https://github.com/angular/angular/pull/59876).

## Usecase

**Fetch** data from an HTTP backend.

Note the emphasis of **fetch**.
It's not meant for mutating data in the backend.
Because an `HttpResourceRef<T>` will be bound to the components/service lifecycle,
ongoing requests will be cancelled, if the component/service gets destroyed.
Also when a new request is emitted, an ongoing HTTP call will be cancelled (`switchMap` behaviour).
Therefore the default HTTP method is `GET`, but other methods are possible, if you need `POST` or `PUT` for _fetching_ data (looking at you, GraphQL 😎).

## Incarnations

Imaging having a component and defining a field inside it.

```ts
@Component({...})
export class UserProfile {
  protected readonly userProfile = httpResource(...);
}
```

In the template you can use the resource as usual.

```
@if (userProfile.hasValue()) {
  @let up = userProfile.value();
  {{ up.Username }}
  <!-- and display other things from the resource -->
} @else if (userProfile.isLoading()) {
  <fancy-spinner />
} @else if (userProfile.error()) {
  <fancy-error [error]="userProfile.error()" />
}
```

The request fires as soon as the `httpResource` is instantiated (or to be more precise, when the effect behind the resource is scheduled).
It will need an injection context to get the `HttpClient` (interceptors will work!).
As usual you can provide an injector if you are running outside an injection context.

The `HttpResourceRef<T>` is a `Resource<T>`, so you have usual suspects regarding signals.

- `.value()`
- `.status()`
- `.isLoading()`
- `.error()`

And functions.

- `.hasValue()`
- `.reload()`

It's also a `WritableResource`.
This means you can overwrite the value of the resource locally.

- `.set()`
- `.update()`
- `.asReadonly()`

New to `HttpResourceRef` are the following signals and methods.

- `.headers()`
- `.progress()`
- `.statusCode()`
- `.destroy()`

### Static URL

```ts
userProfiles = httpResource("/api/user-profiles");
```

### Dynamic URL

```ts
// input, computed, signal, model, ...
selectedUserId: Signal<string> = ...;

// like "computed"
userProfile = httpResource(() => `/api/user-profiles/${this.selectedUserId()}`);
```

### Full Control

Return a `HttpResourceRequest` with different (some optional) properties.
You will know them from the `HttpClient`.

```ts
userProfile = httpResource(() => ({
  method: 'POST',
  url: `/api/user-profiles/${this.selectedUserId()}`,
  headers: ...,
  params: ...,
  body: ...,
  reportProgress: true,
  withCredentials: true,
  transferCache: ..., // Configures the server-side rendering transfer cache for this request.
}));
```

Remember: You shouldn't use `httpResource` for sending updates/mutations to the backend.
You can use this overload, if you need fine grained control over the request for **fetching** data and your backend is "weird".

## Options

As a second parameter you can provide a `HttpResourceOptions` object.

```ts
export interface HttpResourceOptions<TResult, TRaw> {
  defaultValue?: NoInfer<TResult>;
  equal?: ValueEqualityFn<NoInfer<TResult>>;
  injector?: Injector;
  map?: (value: TRaw) => TResult;
}
```

You may know what to do with the properties `defaultValue`, `equal` and `injector`.

### Map the response

Per default the `HttpResourceRef` is typed with `unknown`.
This is because "you never know what you get" from the "box of chocolates" backend (ask Forrest Gump).
Luckily there are libraries which can parse unknown JSON data and provide a typesafe object.
Explore [valibot](https://valibot.dev/) or [zod](https://zod.dev/).

These libraries provide some "parse" function, which returns a typesafe object.
And the `map` option is the right place to plug in those function.
With this in place, the TypeScript compiler will make sure the types are all right.
And you can be sure, that you get what you expect.

## My backend doesn't deliver JSON!

The Angular team has you covered!

```ts
userProfile = httpResource.text(...);
userProfile = httpResource.blob(...);
userProfile = httpResource.arraybuffer(...);
```

This will not try to parse the content of your response as JSON, but will provide you the plain text, a blob or arraybuffer.
That should cover everything you may need.
Especially with the `map` function in place.

Can't wait for the release! 🥰

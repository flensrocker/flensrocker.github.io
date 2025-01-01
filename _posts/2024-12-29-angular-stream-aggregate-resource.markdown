---
layout: post
title:  "Angular: stream aggregate resource"
date:   2024-12-29 14:00:00 +0200
categories: angular resource
---

*Draft In Progress*

## Let's build a custom resource!

> Disclaimer: Written when Angular was at version 19.0.5

The new experimental [Resource API](https://angular.dev/api/core/resource) (at point of writing this) is something I really like to play around with.
It's similar to my ["Signalize async service calls" example]({% link _posts/2024-09-16-angular-signalize-async-service-calls.markdown %}) which I use every day
(even before there were signals, when we used the [`AsyncPipe`](https://angular.dev/api/common/AsyncPipe) to pull the current state into the template).

But more important than the [`resource`](https://angular.dev/api/core/resource) function is the [`Resource<T>` type](https://angular.dev/api/core/Resource) as the base of all APIs which produces some kind of "resource".
And because the `resource` and even the [`rxResource`](https://angular.dev/api/core/rxjs-interop/rxResource) function only accepts "reactive functions" (think "signal getters") as source of the request they are a bit limited in usage.
But they are new and experimental - which means we have to experiment and find out how they can be improved.
And we can expect that they improve over time!

These are the two limitations I want to talk about:

- [`(Rx)ResourceOptions.request`](https://angular.dev/api/core/rxjs-interop/RxResourceOptions#request) which is only a "reactive function that returns the next request".
- [`RxResourceOptions.loader`](https://angular.dev/api/core/rxjs-interop/RxResourceOptions#loader) which uses an `Observable` for the response, but only the first value is used.

In my applications often the frontend connects to the backend via a websocket so it can receive updates whenever another user is mutating some state.
So the "pure" resource doesn't really fit.
Or a simpler example can be an HTTP request with progress events.
Currently the resource would only receive the first progress event and then cancels the stream - not what we want or expect.

But the `Resource<T>` type seems to be the right interface.
If I create an own `resource` function it should return this type, because every Angular developer will know how to handle that in a template.

## Where we start

It all starts with a request.
But not only with a single request, but a stream of requests (of course the stream can be short and can contain only one request).
For now the most commonly used way to describe such a stream is an [(rxjs) observable](https://rxjs.dev/api/index/class/Observable).
I will use the "stream notation" with the dollar sign to distinguish the stream from the single request object.

```ts
const request$: Observable<TRequest>;
```

And then we have a "loader" which transforms each request into a stream of responses.

```ts
type LoaderFn = (request: TRequest) => Observable<TResponse>;
```

These are the first two properties of our `StreamAggregateResourceOptions`.

The core of processing such a request stream (especially when "loading" instead of "posting") is the usual [`switchMap`](https://rxjs.dev/api/index/function/switchMap).
Whenever a new request gets emitted the `loader` function will switch to a new stream of responses.

```ts
const response$: Observable<TResponse> = request$.pipe(
  switchMap((request) => loader(request))
);
```

Don't worry, we will add error handling etc. later, because I don't like too simple examples, where I have to add all those bits before I can actually use it in production.
And error handling is "built in" in the `Resource<T>` - so we don't get around it anyway.

## Where we want to end

We want to produce a `Resource<T>`, which looks like this:

```ts
enum ResourceStatus {
  Idle,
  Error,
  Loading,
  Reloading,
  Resolved,
  Local,
}

interface Resource<T> {
  readonly value: Signal<T | undefined>;
  readonly status: Signal<ResourceStatus>;
  readonly error: Signal<unknown>;
  readonly isLoading: Signal<boolean>;
  hasValue(): boolean;
  reload(): boolean;
}
```

I think it's rather self-explanatory.
Of course everything is a `Signal` because it can change during the lifetime of the resource.
And signals are "the way" to track (and react to) changes.

The `value` contains the current (or last) value of the resource.
It can be `undefined`, because we start with an "empty" resource before any request got emitted or returned a response.
I will get back to this later, when I introduce the `initialValue` of my "stream aggregate resource" implementation.

The `status` is the detailed information about the state the resource is currently in.
Derived from that there's the `isLoading` signal, which should be `true` when the resource is in `Loading` or `Reloading` state.

And the `error` will contain the last error, whatever that will be.
We will use this for our error handling of our observable stream, because you always want to handle errors in your streams otherwise they stop working.

The `hasValue()` function is a helper that we can use inside the template to only include the part of the view, which displays the resource's value, when we actually have a value.

And finally the `reload()` function enables us to provide some kind of reload mechanism, which we might want to trigger in some circumstances.
This is the only property of `Resource<T>` which I'm not really convinced of, that it's included in this type.
We're allowed to ignore it - if a reload can't be triggered, this function just can return `false` and do nothing.
But of course, we will get to it later, too - because implementing a reload behavior in an observable stream is not that difficult...

## Connecting the dots

The `ResourceStatus` enum is our map to follow from start to end.
For each kind of state we'll think about how the stream can get into that state.

In the end we will hopefully get a stream of objects with a `status`, `value` and optional `error`.
Because every other field in the `Resource<T>` can be derived from these three.

And to bridge the gap between the `Observable` to the `Signal` world,
we will just subscribe to the observable and push every new value into a writeable signal,
which we'll expose in the shape of an `Resource<T>`.

### Idle (and Resovled)

This is pretty easy.
It's the state the resource starts in.
So we have to think about its initial state regarding the other fields.

```ts
{
  status: ResourceStatus.Idle,
  value: undefined,
  error: undefined,
}
```

There's one thing I don't really like.
If this is our initial state, then we have to deal with `undefined` values in the template.
You can always do that with something like

```html
@if (resource.value(); as val) {
  <!-- use val inside this block -->
} @else {
  <!-- optionally show some "idle" content -->
}
```

But most of the time, I will write something like this in the component to avoid `undefined` in the template:

```ts
protected readonly value = computed(() => {
  const value = this.resource.value();
  return value ?? {
    // some initial value like an empty array when displaying a list of item etc.
  };
});
```

And rumors say the Angular team is considering some kind of ["initial" or "default" value](https://github.com/angular/angular/issues/58840#issuecomment-2500226764).

Since the purpose of this custom resource is to aggregate values over time like
[`scan`](https://rxjs.dev/api/index/function/scan) or [`Array.prototype.reduce`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce) do
we will need some initial value anyway - so let's add that to our `StreamAggregateResourceOptions`.

We will also distinguish the actual shape of the resource `TResource` from the kind of values `TResponse` our loader returns.
With that we get some flexibility inside our aggregate function.

And we will use the "undefined" request as a trigger to re-enter the `Idle` state, so we add that to the type definition.

```ts
type StreamAggregateResourceOptions<TResource, TRequest, TResponse = TResource> = {
  readonly initialValue: TResource;
  readonly request: Observable<TRequest | undefined>;
  readonly loader: (request: NoInfer<TRequest>) => Observable<TResponse>;
  readonly aggregate: (accumulator: TResource, current: TResponse) => TResource;
};
```

With this we can build the first skeleton of our stream.

```ts
// the signal we will push our stream values into
const source = signal({
  status: ResourceStatus.Idle,
  value: options.initialValue,
  error: undefined,
});

// derived signals from which we will build our Resource<TResource>
const status = computed(() => source().status);
const value = computed(() => source().value);
const error = computed(() => source().error);
const isLoading = computed(() => status() === ResourceStatus.Loading || status() === ResourceStatus.Reloading);
const hasValue = () => true; // our resource will always have a value!
const reload = () => false; // no reload functionality yet

request$.pipe(
  switchMap((request) => {
    if (request === undefined) {
      return of({
        status: ResourceStatus.Idle,
        // preserve the last "value" of the resource
        value: undefined,
        // clear any old error
        error: undefined,
      });
    }

    return loader(request).pipe(map(response => ({
      status: ResourceStatus.Resolved,
      value: response,
      error: undefined,
    })));
  }),
)
.subscribe({ // TODO Think about unsubscribe!
  next: next => {
    source.update(previous => ({
      status: next.status,
      value: next.value ? options.aggregate(previous.value, next.value) : previous.value,
      error: next.error,
    }));
  },
});
```

So we accidentally also covered the `Resolved` state... 🤷

### Loading

`TODO`

### Error

`TODO`

### Reloading

`TODO`

### Local

It's possible to extend this example with a `Local` functionality.
But I haven't needed it yet, so if you do - extend!

### Don't forget to unsubscribe

`TODO`

## Summary

`TODO`

...to be continued...

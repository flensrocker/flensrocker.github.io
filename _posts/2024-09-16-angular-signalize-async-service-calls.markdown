---
layout: post
title:  "Angular: Signalize async service calls"
date:   2024-09-16 18:00:00 +0200
categories: angular signals
---

Almost every Angular application has to deal with some kind of asynchronous calls:

```typescript
export type ServiceCallFn<TRequest, TResponse> =
  (request: TRequest) => Observable<TResponse>;
```

It can be an HTTP call (GET, POST etc.), websockets, IndexedDB, mouse events, promises - you name it.

Now that signals become popular and everyone tries to avoid the [`AsyncPipe`](https://angular.dev/api/common/AsyncPipe), how can we "signalize" such a call?

> _Disclaimer:_
> This is what I do and it works in my context.
> I'm sure that you are able to adopt my concepts and modify them as **you** need them to be.

My main usecase of signals in Angular apps is to display "state" in a template:
That can be forms, fetched data, a progress indicator, error messages and everything in between.

Looking at the typical "HTTP GET" the template is always in one of the following states:

- `IDLE` - no request has been sent yet.
- `BUSY` - a request was generated and the service gets called (I avoid the term "loading" here, because it doesn't fit other calls like "HTTP POST").
- `SUCCESS` - a response came back.
- `ERROR` - instead of a response we got an error.

(the other kind of service calls all follow the same schema - they all transition between these states.)

Translating these states to TypeScript they can be expressed like this:

```typescript
export type ServiceCallIdle = {
  readonly type: 'IDLE';
};

export type ServiceCallBusy<TRequest> = {
  readonly type: 'BUSY';
  readonly request: TRequest;
};

export type ServiceCallSuccess<TRequest, TResponse> = {
  readonly type: 'SUCCESS';
  readonly request: TRequest;
  readonly response: TResponse;
};

export type ServiceCallError<TRequest> = {
  readonly type: 'ERROR';
  readonly request: TRequest;
  readonly error: unknown;
};

export type ServiceCall<TRequest, TResponse> =
  | ServiceCallIdle
  | ServiceCallBusy<TRequest>
  | ServiceCallSuccess<TRequest, TResponse>
  | ServiceCallError<TRequest>;
```

When I design such states with discriminated unions, I like to add some "create" helper functions.
They will make the following code more readable.

```typescript
const idleServiceCall: ServiceCallIdle = {
  type: 'IDLE',
};

const busyServiceCall = <TRequest>(
  request: TRequest
): ServiceCallBusy<TRequest> => ({
  type: 'BUSY',
  request,
});

const successServiceCall = <TRequest, TResponse>(
  request: TRequest,
  response: TResponse
): ServiceCallSuccess<TRequest, TResponse> => ({
  type: 'SUCCESS',
  request,
  response,
});

const errorServiceCall = <TRequest>(
  request: TRequest,
  error: unknown
): ServiceCallError<TRequest> => ({
  type: 'ERROR',
  request,
  error,
});
```

At this point in my design process I wonder what kind of (typesafe) state my template is going to need.
I'm not an UX designer but you get, what I mean...

{% raw %}
```html
@if (foo.idle()) {
  <div>
    Waiting for incoming request...
  </div>
}

@if (foo.busy(); as busy) {
  <div>
    Busy on Request: {{ busy.request | json }}
  </div>
}

@if (foo.success(); as success) {
  <div>
    Request: {{ success.request | json }}
  </div>
  <div>
    Response: {{ success.response | json }}
  </div>
}

@if (foo.error(); as error) {
  <div>
    Request: {{ error.request | json }}
  </div>
  <div>
    Error: {{ error.error | json }}
  </div>
}
```
{% endraw %}

So it looks like the state I want to generate looks something like this:

```typescript
export type ServiceCallState<TRequest, TResponse> = {
  readonly idle: Signal<boolean>;
  readonly busy: Signal<Omit<ServiceCallBusy<TRequest>, 'type'> | false>;
  readonly success: Signal<
    Omit<ServiceCallSuccess<TRequest, TResponse>, 'type'> | false
  >;
  readonly error: Signal<Omit<ServiceCallError<TRequest>, 'type'> | false>;
};
```

Executing the service call with some standard [rxjs](https://rxjs.dev/) operators should be pretty straight forward.
But we want to choose if we `switchMap` or `concatMap`
(switching on a "HTTP POST" request, when you want to save something, is not the behavior your users want).

```typescript
export type ServiceCallOptions = {
  readonly behavior: 'SWITCH' | 'CONCAT';
};

const defaultServiceCallOptions: ServiceCallOptions = {
  behavior: 'SWITCH',
};

const setupServiceCall = <TRequest, TResponse>(
  $request: Signal<TRequest> | Observable<TRequest>,
  serviceFn: ServiceCallFn<TRequest, TResponse>,
  options: ServiceCallOptions
): Observable<ServiceCall<TRequest, TResponse>> => {
  const request$ = isSignal($request) ? toObservable($request) : $request;
  const mapFn = options.behavior === 'CONCAT' ? concatMap : switchMap;

  return request$.pipe(
    mapFn((request) =>
      serviceFn(request).pipe(
        map((response) => successServiceCall(request, response)),
        catchError((error) => of(errorServiceCall(request, error))),
        startWith(busyServiceCall(request))
      )
    ),
    // cancel the service call when the calling component gets destroyed
    takeUntilDestroyed(),
    // if we have multiple subscriber, call the service only once
    share()
  );
};
```

Putting the pieces together:

```typescript
export const serviceCall = <TRequest, TResponse>(
  $request: Signal<TRequest> | Observable<TRequest>,
  serviceFn: ServiceCallFn<TRequest, TResponse>,
  options?: ServiceCallOptions
): ServiceCallState<TRequest, TResponse> => {
  options = {
    ...defaultServiceCallOptions,
    ...options,
  };
  const state$ = setupServiceCall($request, serviceFn, options);

  const $state = toSignal(state$, {
    initialValue: idleServiceCall,
  });

  return {
    idle: computed(() => $state().type === 'IDLE'),
    busy: computed(() => {
      const state = $state();
      if (state.type !== 'BUSY') {
        return false;
      }

      const { type, ...busy } = state;
      return busy;
    }),
    success: computed(() => {
      const state = $state();
      if (state.type !== 'SUCCESS') {
        return false;
      }

      const { type, ...success } = state;
      return success;
    }),
    error: computed(() => {
      const state = $state();
      if (state.type !== 'ERROR') {
        return false;
      }

      const { type, ...error } = state;
      return error;
    }),
  };
};
```

You can find [the code on StackBlitz](https://stackblitz.com/edit/stackblitz-starters-vjxdge?file=src%2Fservice-call.ts).

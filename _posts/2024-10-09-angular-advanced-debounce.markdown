---
layout: post
title:  "Angular Forms: advanced debounce"
date:   2024-10-09 20:00:00 +0200
categories: angular forms
---

We all have done a "debounced search form" at some point in our life.
Given a text input, when the user types something into it,
then after a (small) amount of time the form value is sent as the search request to the backend to fetch some results.
If the user changes the input the ongoing request is cancelled (if it's still on the fly ) and a new one is started.

It's the famous combination of [`debounceTime`](https://rxjs.dev/api/index/function/debounceTime)
and [`switchMap`](https://rxjs.dev/api/index/function/switchMap)
on the [`AbstractControl.valueChanges`](https://angular.dev/api/forms/AbstractControl#valueChanges) we all know and love.

But there's one thing which bothers me on such search forms.
If you not only have a text input but some additional options like checkboxes or selects,
than changes on one of those are also debounced.
Or I would really like to "Enter" after I finished typing my search term to trigger the request without waiting for the debounce to happen.

Lucky for us rxjs has us covered with the [`debounce`](https://rxjs.dev/api/index/function/debounce) operator.
And that combined with the new [`AbstractControl.events`](https://angular.dev/api/forms/AbstractControl#events) provides the right tools to improve our users' experience.

## The Simple Implementation

There are two interesting events:

- [`ValueChangeEvent`](https://angular.dev/api/forms/ValueChangeEvent)
- [`FormSubmittedEvent`](https://angular.dev/api/forms/FormSubmittedEvent) (I already used that in the ["declarative submit" post]({% link _posts/2024-09-21-angular-forms-declarative-submit.markdown %}))

Both are emitted from the `events` observable, so we should be able to debounce only on `ValueChangeEvent`s and let the `FormSubmittedEvent` just pass without debounce.

For these kind of usecases the `debounce` operator accepts a function, which returns an observable depending on the current value of the source observable.
If we want to debounce, we have to return an observable which emits in the future, e.g. with the [`timer`](https://rxjs.dev/api/index/function/timer) function.
If we don't want do debounce, we can return an observable which emits synchronously like [`of`](https://rxjs.dev/api/index/function/of).

The first implementation could be something like this:

```typescript
const debounceMs = 500;

form.events.pipe(
  filter(
    (event) =>
      event instanceof ValueChangeEvent || event instanceof FormSubmittedEvent
  ),
  debounce((event) =>
    event instanceof FormSubmittedEvent ? of(0) : timer(debounceMs)
  ),
  map(() => form.getRawValue()),
  switchMap((query) => service.search(query)),
  // etc.
);
```

And it will totally do the job.
Of course you have to add some error handling on the service call.
And of course we don't want to emit invalid queries.
And what should happen, if the user modifies the input to an invalid state - should the old results be visible or cleared?

Time for...

## The Advanced Implementation

(aka over-engineering 😎)

We want:

- filter for `ValueChangeEvent` and `FormSubmittedEvent`,
- map that to a "value with reason" which can be one of `DEBOUNCE`, `SUBMIT` or `NOT_VALID`,  
  (If the source of the `ValueChangeEvent` is not a "control to be debounced" the reason also will be mapped to `SUBMIT`.
  The "controls to be debounced" will be provided as a list of control paths like [`AbstractControl.get()`](https://angular.dev/api/forms/AbstractControl#get) would accept.)
- debounce the emission on the reason `DEBOUNCE`,
- map the reason `NOT_VALID` to `null` otherwise pass the form value,
- since all `NOT_VALID`s are mapped to `null` we add a [`distinctUntilChanged`](https://rxjs.dev/api/index/function/distinctUntilChanged),
- and because we don't know how many subscribers there will be, we [`share`](https://rxjs.dev/api/index/function/share) the observable.

After that we can feed these values into something which handles [async service calls]({% link _posts/2024-09-16-angular-signalize-async-service-calls.markdown %}).

Done! [Here's the code on Stackblitz](https://stackblitz.com/edit/stackblitz-starters-pki3uu?file=src%2Fdebounce-form.ts).

(There's also some simple paging involved - but that's not important for the debounce part of this example.)

## The Details

Beside the [`FormValueOf` helper type]({% link _posts/2024-09-18-angular-utils-form-value-of.markdown %}) we need some more helpers.

### ValueSource

A "source of values" can either be just a value, a signal or an observable.
All these can be converted into an observable.
This enables a more reactive approach, so we can not only start from a given form but also from signals or observables containing forms.
If we want to be "reactive from end to end" we should enable the users of our utilities to chain them into whatever reactive stream they have.

```typescript
import { Injector, Signal, isSignal } from '@angular/core';
import { toObservable } from '@angular/core/rxjs-interop';

import { Observable, isObservable, of } from 'rxjs';

export type ValueSource<TValue> = TValue | Signal<TValue> | Observable<TValue>;

export type ValueSourceOptions = Readonly<{
  injector?: Injector;
}>;
/**
 * Converts any `ValueSource` into an observable.
 * Usefull to create observable streams from any value source.
 */
export const sourceToObservable = <TValue>(
  source: ValueSource<TValue>,
  options?: ValueSourceOptions
): Observable<TValue> =>
  isObservable(source)
    ? source
    : isSignal(source)
    ? toObservable(source, { injector: options?.injector })
    : of(source);
```

### getControlPathFromRoot

When we process a `ValueChangeEvent` the `source` property reflects which control triggered this event.
This can be any control nested deep into the whole `FormGroup`.

Angular doesn't provide a function (correct me if I'm wrong), which returns the path from the root form up to the given control.
It's not that difficult - we just have to search through the controls of the parent control until we find it and get the name.
And do that up to the root control (recursion!).
And then we join all those names separated with a dot.
Ah - don't forget about `FormArray`s - their `controls` is an array with numbers as keys.

```typescript
import { AbstractControl } from '@angular/forms';

type ControlName = string | number;
type ControlPath = readonly ControlName[];
const EmptyControlPath: ControlPath = [];

const controlPathToFormPath = (path: ControlPath): string => {
  return path == null || path.length === 0 ? '' : path.join('.');
};

const getControlName = (control: AbstractControl): ControlName | null => {
  if (control.parent == null) {
    return null;
  }

  const children = control.parent.controls;
  if (Array.isArray(children)) {
    for (let index = 0; index < children.length; index++) {
      if (children[index] === control) {
        return index;
      }
    }
    return null;
  }

  return (
    Object.keys(children).find((name) => control === children[name]) ?? null
  );
};

const getControlPath = (control: AbstractControl | null): ControlPath => {
  let path: ControlPath = EmptyControlPath;

  let current: AbstractControl | null = control;
  while (current != null) {
    const name = getControlName(current);
    if (name == null) {
      break;
    }

    path = [name, ...path];
    current = current.parent;
  }

  return path;
};

/**
 * Returns the path of the control from the root to itself.
 * Can be used by `.get(...)` on the root.
 */
export const getControlPathFromRoot = (
  control: AbstractControl | null
): string => {
  return controlPathToFormPath(getControlPath(control));
};
```

(I told you we over-engineer!)

### debounceForm

And finally - the code!

```typescript
import {
  AbstractControl,
  ControlEvent,
  FormSubmittedEvent,
  ValueChangeEvent,
} from '@angular/forms';
import {
  Observable,
  debounce,
  distinctUntilChanged,
  filter,
  map,
  of,
  share,
  switchMap,
  timer,
} from 'rxjs';

import { FormValueOf } from './form-value-of';
import { getControlPathFromRoot } from './get-control-path-from-root';
import { sourceToObservable, ValueSource } from './value-source';

type ValueAndReason<TValue> = Readonly<{
  value: TValue;
  reason: 'NOT_VALID' | 'DEBOUNCE' | 'SUBMIT';
}>;

const filterForValueChangeAndSubmittedEvents = () =>
  filter(
    (event: ControlEvent) =>
      event instanceof ValueChangeEvent || event instanceof FormSubmittedEvent
  );

const mapToValueAndReason = <TControl extends AbstractControl>(
  form: TControl,
  debounceOnSet: ReadonlySet<string>
) =>
  map(
    (
      event: ValueChangeEvent<FormValueOf<TControl>> | FormSubmittedEvent
    ): ValueAndReason<FormValueOf<TControl>> => ({
      value: form.getRawValue(),
      reason:
        form.status !== 'VALID'
          ? 'NOT_VALID'
          : debounceOnSet.has(getControlPathFromRoot(event.source))
          ? 'DEBOUNCE'
          : 'SUBMIT',
    })
  );

type DebounceIfPredicate<T> = (value: T) => boolean;

const debounceIf = <T>(predicate: DebounceIfPredicate<T>, debounceMs: number) =>
  debounce((value: T) => (predicate(value) ? timer(debounceMs) : of(0)));

const mapToValidValueOrNull = <TValue>() =>
  map(({ reason, value }: ValueAndReason<TValue>) =>
    reason === 'NOT_VALID' ? null : value
  );

/**
 * Creates an observable which emits the values of the form
 * whenever it changes or is submitted.
 *
 * It emits `null` if the status of the whole form is not `VALID`.
 *
 * When the name of the control, which is responsible for the `ValueChangeEvent`,
 * is included in the `debounceOn` array, the emission of the form value will be
 * debounced by the time given in `debounceMs`.
 *
 * The `debounceOn` array can include dot separated paths to nested controls
 * like in `AbstractControl.get('nested.property')`.
 */
export const debounceForm = <TControl extends AbstractControl>(
  $form$: ValueSource<TControl>,
  debounceMs: number,
  ...debounceOn: string[]
): Observable<FormValueOf<TControl> | null> => {
  const debounceOnSet = new Set<string>(debounceOn);

  return sourceToObservable($form$).pipe(
    switchMap((form) =>
      form.events.pipe(
        filterForValueChangeAndSubmittedEvents(),
        mapToValueAndReason(form, debounceOnSet),
        debounceIf(({ reason }) => reason === 'DEBOUNCE', debounceMs),
        mapToValidValueOrNull(),
        distinctUntilChanged()
      )
    ),
    share()
  );
};
```

That was fun! 🥰

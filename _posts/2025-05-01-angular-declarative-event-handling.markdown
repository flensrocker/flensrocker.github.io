---
layout: post
title: "Angular: declarative event handling"
date: 2025-05-01 13:00:00 +0200
categories: angular declarative event
---

There's always some discussion about "declarative vs. imperative" programming - and both are right.

In the Angular context, if you keep your components small enough, it doesn't really matter, if you code in declarative or imperative style.
The usual argument of the declarative people is, that you only have to look at one place to understand, what's happening (the declaration).
But, as mentioned, if your component is small enough, you may also understand, what's going on, looking at "one place" (which might be more than a few lines, but fits on a screen).

After all, I really like declarative programming.
I find the mental model behind this easier to understand.

With the latest updates on signals etc. Angular gets more and more declarative.
But there are gaps not filled yet, where you have to do some imperative steps.
For example one of this are consuming button clicks and trigger some sideeffect like fetching some data etc.

## Imperative

What you often see is, that you declare a rxjs subject like

```ts
protected readonly clicked$ = new Subject<void>();
```

in your component,
and let the button's click handler call the `next` methond on that subject:

```html
<button type="button" (click)="clicked$.next()">Click me!</button>
```

> _Note:_ I added the not really neccessary, but in my opinion, important modifiers `protected` and `readonly`,
> because you don't want to expose this property outside of your component
> and you don't want to reassign it with another subject instance,
> because that would break derived observables.

Having this in place, you can then use the declarative paradigm to define whatever should happen, when the button is clicked.
You can map the click to a request and feed it into some kind of resource, to let it fetch some data.

Here's a little example with the least amount of error handling
(not handling errors is not an option, because the observable then stops working).

```ts
type Data<T> = Readonly<{
  value?: T;
  error?: unknown;
  loading?: true;
}>;

@Injectable()
export class SomeService {
  getData(): Observable<string> {
    return of('the value').pipe(
      delay(1000),
      map((v) => {
        if (Math.random() < 0.5) {
          throw 'the error';
        }

        return v;
      })
    );
  }
}

@Component({ ... })
export class SomeComponent {
  readonly #someService = inject(SomeService);

  protected readonly clicked$ = new Subject<void>();

  protected readonly data$: Observable<Data<string>> = this.clicked$.pipe(
    switchMap(() => this.#someService.getData().pipe(
      map((value) => ({ value } satisfies Data<string>)),
      catchError((error) => of({ error } satisfies Data<string>)),
      startWith({ loading: true } satisfies Data<string>)
    ))
  );
}
```

And a simple template, just to get the idea.

```html
<button type="button" (click)="clicked$.next()">Click me!</button>

<div>
{% raw %}
  @if (data$ | async; as data) {
    @if (data.loading) {
      loading...
    } @else if (data.error) {
      {{ data.error | json }}
    } @else {
      {{ data.value }}
    }
  }
{% endraw %}
</div>
```

## Declarative

So, how can we consume the "click" in a declarative way?

Angular and rxjs provide the building blocks, but it's a bit rough and not really beautiful...

- We will need the button: `viewChild`
- We will need the event: `fromEvent`

(I really wish, Angular would provide some kind of `viewChildEvent` function, which returns either an `Observable` or even an `OutputRef`, which is nearly the same)

Create some helper function, to make it easier consuming the event.

```ts
export const toEvent = <TEvent extends Event = Event>(
  elementRef: Signal<ElementRef | null | undefined>,
  eventName: string,
  injector?: Injector
): Observable<TEvent> => {
  const elementRef$ = toObservable(elementRef, { injector });
  return elementRef$.pipe(
    switchMap((el) =>
      el == null ? NEVER : fromEvent<TEvent>(el.nativeElement, eventName)
    )
  );
};
```

Name the button in the template with something like `#btn`.

```html
<button #btn type="button">Click me!</button>
```

And retrieve and use it in the component.

```ts
protected readonly btn = viewChild("btn", { read: ElementRef });

protected readonly data$ = toEvent(this.btn, "click").pipe(...);
```

## Summary

What's the difference between these two approaches?

In the imperative way you have to

- declare a subject in TS
- trigger the subject from the template

In the declarative way you have to

- name the button in the template
- retrieve the button from the template
- get the event from the button

After that, in both scenarios you have an observable, which you can use to build a reactive pipeline.

I left out a little detail:
With the subject you can easily feed some data from the template into it, which you then can use in your pipeline.

With the `toEvent` function it needs some extra lifting.
What you can do, is to set the data to some `data-thing` attribute on the button
and get its value from the `event.target.dataset["thing"]`.
And if you need that on a regular basis you can incorporate that into the `toEvent` function.

I find the declarative approach a little bit nicer, because there's less logic in the template.
But after all both ways are not that different from each other.
So use, whatever you and your team are comfortable with!

You can find the code in [this StackBlitz](https://stackblitz.com/edit/stackblitz-starters-y445jnks?file=src%2Fmain.ts).

Followup exercises for the reader:

- Create a `toEvents` function, which can consume a `Signal<readonly ElementRef[]>` returned by `viewChildren` and merge all events into one stream.
- Adapt this concept to consume from outputs of components.

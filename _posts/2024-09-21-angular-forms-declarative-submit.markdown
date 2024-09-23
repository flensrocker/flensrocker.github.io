---
layout: post
title:  "Angular Forms: declarative submit"
date:   2024-09-21 16:00:00 +0200
categories: angular forms
---

I like declarative programming.
Combined with immutable data a lot of problems just don't exist - but of course you get others...
It's another kind of thinking, the so called "mind shift".

I started using [Angular](https://angular.dev/) with version 2.0 after some minor steps with AngularJS.
After the usual struggle to learn [RxJs](https://rxjs.dev) I appreciated the mental model the observables taught me.
And since my main application is an ERP with lots of forms I took a header into reactive forms.

Using the `valueChanges` and `statusChanges` observables with some of the common operators like `map`, `switchMap` or `filter` and `debounceTime` was pretty straightforward.
At least for the simple, more common usecases.
But what annoyed me every time was how I have to handle submitting a form.

Assuming that the form's value should be sent via something like an HTTP POST request you have different choices how to handle that.
In the template you can register some function with `(ngSubmit)` on the root form element.
And in the component you then have to either start some observable and manual subscribe to it (🙈),
or push the current form value into some subject so it can be handled by some pipeline subscribed by an `AsyncPipe` in the template to show the progress or errors.
This is an imperative step I wanted to avoid - trying several things with querying the `FormGroupDirective` from the template etc.
No solution really pleased me.

Finally with Angular 18 the [proposal #10887](https://github.com/angular/angular/issues/10887) was merged and we got _"One Observable To Rule Them All"_!
And one of all the events which get emitted by `AbstractControl.events` is the [`FormSubmittedEvent`](https://angular.dev/api/forms/FormSubmittedEvent) - yay!

So the first thing I had to do was to write this little helper:

```typescript
export const validFormSubmit = <
  TControl extends {
    [K in keyof TControl]: AbstractControl<unknown>;
  },
>(
  form: FormGroup<TControl> | Signal<FormGroup<TControl>>
): Observable<FormValueOf<FormGroup<TControl>>> => {
  const formEvents = isSignal(form)
    ? toObservable(form).pipe(switchMap(f => f.events))
    : form.events;

  return formEvents.pipe(
    filter(controlEvent =>
      controlEvent instanceof FormSubmittedEvent
      && controlEvent.source.status === "VALID"),
    map(submittedEvent => submittedEvent.source.getRawValue())
  );
};
```

(You can find the source for that `FormValueOf` type in [another post]({% link _posts/2024-09-18-angular-utils-form-value-of.markdown %}).)

It just filters the control events for the `FormSubmittedEvent` and checks if the form is valid.
And that gets mapped to the (raw) value of the given form.
That's it!

(You can ignore that signal stuff at the beginning - sometimes I have signals which holds the `FormGroup`, so I want to handle such cases.)

Now I have an observable which emits whenever the form gets submitted **AND** its value is `VALID`.
Add a `map` or whatever it needs to transform it into the request your service needs
and then you can use something like the [`serviceCall` helper]({% link _posts/2024-09-16-angular-signalize-async-service-calls.markdown %})
to declarative handle the form submit.

That was fun and made me happy - and it makes me happy every time I use it.

Here's some [StackBlitz code](https://stackblitz.com/edit/stackblitz-starters-mnrnee?file=src%2Fmain.ts) to play with it.

BTW: You know the difference between "not `VALID`" and `INVALID`?
It's `DISABLED | PENDING`...  
You have to take that into account when you want your submit button to be disabled if the form is "not valid".

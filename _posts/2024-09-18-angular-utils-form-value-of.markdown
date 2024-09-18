---
layout: post
title:  "Angular Utils: FormValueOf<T>"
date:   2024-09-18 20:00:00 +0200
categories: angular utils forms
---

With Angular 14 the team introduced [Typed Forms](https://angular.dev/guide/forms/typed-forms).
As I work with a lot of forms in my applications it was a great relief to finally have the right tools to use them in a proper way.
The main forms are refactored to typed ones, but there are always some `UntypedFormGroup`s left...

But there is some small piece I miss - so I have to build it by myself!

As a dotnet developer I'm used to "type all the things".
And in my TypeScript code I often write more types than necessary.
But I really like to decorate the arguments and return values of my functions.

What I needed was a derived type of the object which represents the form's value.
That type you get infered, when you call `form.getRawValue()`
(I don't want to talk about `form.value` and `null` and all the stuff - it's there and it cannot go away easily because backward compatibility).

Given a form like this:

```typescript
const form = new FormGroup({
  foo: new FormControl<string>('', { nonNullable: true }),
  sub: new FormGroup({
    prop: new FormControl<number>(0, { nonNullable: true }),
  }),
});
```

I want to use values of such a form, e.g. in a transformation function.

```typescript
type SomeOtherType = {
  readonly foo: string;
  readonly subProp: number;
};

const transform = (formValue: ???): SomeOtherType => {
  return {
    foo: formValue.foo,
    subProp: formValue.sub.prop,
  };
};
```

Of course I could manually generate the right type:

```typescript
type FormValue = {
  foo: string;
  sub: {
    prop: number;
  };
};
```

But we're TypeScript developers and we should use what's available!
This is a fantastic usecase for [Mapping Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html).

So, what should the mapping do?

- We want `FormControl<T>` to map to a plain property of type `T`.
- A `FormGroup<T>` should map to an object and all controls of that group should be properties with the right type.
  Some will be `FormControl`s, but we can have nested `FormGroup`s - I smell some recursion...
- `FormRecord`s are just fancy `FormGroup`s so we can ignore them.
- And `FormArray<T>` should map to... an array of course!
  The type of the items should be derived in a similar way as above.

All of those form types derive from `AbstractControl`.
So that will be our starting point.

After that we have to think a little about the order of our mappings but after every pice just falls automagically at the right place.
The [`infer` keyword of conditional types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html) plays an important role.

And here it is!

```typescript
export type FormValueOf<T extends AbstractControl> = T extends FormControl<
  infer TControl
>
  ? TControl
  : T extends FormGroup<infer TGroup>
  ? { [K in keyof TGroup]: FormValueOf<TGroup[K]> }
  : T extends FormArray<infer TElement>
  ? FormValueOf<TElement>[]
  : T;
```

It works as follows:

- If the provided type extends a `FormControl<TControl>` infer its type parameter `TControl` and just map to that type.
- Else if the provided type extends a `FormGroup<TGroup>` map that type (which must be an object type) to an object with the same properties.
  And the type of each property is the `FormValueOf<TGroup[K]>` of that control in the form group.
- Else if the type is a `FormArray<TElement>` then map the elements' type (which is some kind of `AbstractControl`) with `FormValueOf` and put it in an array.
- The last case "should never happen", but if the Angular team introduce a new type of control we just map it to itself.
  Since we follow the release notes of every Angular release we can extend our mapping when we upgrade.

And this is the way I like to use it:

- Define the type of the form.
- Derive the values type.
- Declare an initial value for the form.
- Write a function which creates the form from that value.
- If form arrays are involved: write a function which sets the value into an existing form with the right number of items in the array.

```typescript
type MyFormItem = FormGroup<{
  id: FormControl<string>;
  name: FormControl<string>;
}>;

type MyForm = FormGroup<{
  foo: FormControl<string>;
  sub: FormGroup<{
    prop: FormControl<number>;
  }>;
  items: FormArray<MyFormItem>;
}>;

type MyFormItemValue = FormValueOf<MyFormItem>;
type MyFormValue = FormValueOf<MyForm>;

const initialMyFormItemValue: MyFormItemValue = {
  id: '',
  name: '',
};
const initialMyFormValue: MyFormValue = {
  foo: '',
  sub: {
    prop: 0,
  },
  items: [],
};

const createMyFormItem = (itemValue: MyFormItemValue): MyFormItem =>
  new FormGroup({
    id: new FormControl<string>(itemValue.id, { nonNullable: true }),
    name: new FormControl<string>(itemValue.name, { nonNullable: true }),
  });

const createMyForm = (formValue: MyFormValue): MyForm =>
  new FormGroup({
    foo: new FormControl<string>(formValue.foo, { nonNullable: true }),
    sub: new FormGroup({
      prop: new FormControl<number>(formValue.sub.prop, { nonNullable: true }),
    }),
    items: new FormArray(formValue.items.map(createMyFormItem)),
  });

const setMyFormValue = (
  form: MyForm,
  formValue: MyFormValue,
  options?: {
    onlySelf?: boolean;
    emitEvent?: boolean;
  }
): void => {
  while (form.controls.items.length > formValue.items.length) {
    form.controls.items.removeAt(form.controls.items.length - 1, {
      emitEvent: false,
    });
  }
  while (form.controls.items.length < formValue.items.length) {
    form.controls.items.push(createMyFormItem(initialMyFormItemValue), {
      emitEvent: false,
    });
  }
  form.setValue(formValue, options);
};

const myForm = createMyForm(initialMyFormValue);

const currentValue = myForm.getRawValue();

currentValue.items.push({
  id: '3',
  name: 'Three',
});

setMyFormValue(myForm, currentValue);
```

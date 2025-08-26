---
layout: post
title: "Angular: Type-Safe Form Control Path"
date: 2025-08-26 10:00:00 +0200
categories: angular forms typescript
---

Did you ever wonder how functions like [`AbstractControl.get(path)`](https://angular.dev/api/forms/AbstractControl#get)
can be made type-safe, so you only can provide strings, that actually point to an existing control?
Even nested?

This is one of my favourite capabilities of TypeScript - mapping types to increase the developer experience for the users of my code (me included!).

## The Problem

We have some nested structure and we want to provide a function, which accepts some "path" into this structure.
We should be able to "dot into" nested objects.
Nested `FormGroup`s are a perfect example for this, but it can be anything.

```ts
const fb: NonNullableFormBuilder = ...;
const form = fb.group({
  name: fb.control<string>(''),
  email: fb.control<string>(''),
  address: fb.group({
    street: fb.control<string>(''),
    streetNo: fb.control<string>(''),
    city: fb.control<string>(''),
    zipCode: fb.control<string>(''),
    country: fb.control<string>(''),
  }),
});
```

Let's try to make the [`debounceOn` argument from the `debounceForm` function]({% link _posts/2024-10-09-angular-advanced-debounce.markdown %}) type-safe,
so that we can only pass valid paths for actual existing controls.

Our input is the type of the form (ignore the `ɵNonNullableFormControls`, it means basically "object with controls"):

```ts
type TheForm = typeof form;
// which resolves to:
type TheForm = FormGroup<ɵNonNullableFormControls<{
  name: FormControl<string>;
  email: FormControl<string>;
  address: FormGroup<ɵNonNullableFormControls<{
    street: FormControl<string>;
    streetNo: FormControl<string>;
    city: FormControl<string>;
    zipCode: FormControl<string>;
    country: FormControl<string>;
  }>>;
}>>
```

What we need is a discriminated union of all the valid control paths:

```ts
type TheFormPath =
  | 'name'
  | 'email'
  | 'address'
  | 'address.street'
  | 'address.streetNo'
  | 'address.city'
  | 'address.zipCode'
  | 'address.country';
```

## This is ~~the~~ one way

Writing "Mapping Types" is always easier than reading them.
The only way to get better at reading them is to practice writing them so you get a feeling, what's going on, if you see something like this:

```ts
type FormControlPath<TControl extends AbstractControl> =
  TControl extends FormGroup | FormArray
    ? {
        [K in keyof TControl['controls']]:
          TControl['controls'][K] extends FormGroup | FormArray
            ? 
              | K
              | `${K & (string | number)}.${FormControlPath<TControl['controls'][K]> & (string | number)}`
            : TControl['controls'][K] extends AbstractControl
              ? K
              : never;
      }[keyof TControl['controls']]
    : never;
```

First let's take some inventory.

All the controls in an Angular Form derive from [`AbstractControl`](https://angular.dev/api/forms/AbstractControl).

- [`FormControl`](https://angular.dev/api/forms/FormControl)
- [`FormGroup`](https://angular.dev/api/forms/FormGroup)
- [`FormArray`](https://angular.dev/api/forms/FormArray)
- [`FormRecord`](https://angular.dev/api/forms/FormRecord)

We can ignore the `FormRecord` because that is derived from `FormGroup`,
so when we handle the `FormGroup` we already handle `FormRecord`s.

Only the `FormGroup` and the `FormArray` have a property `controls`, where we can find the child controls.
We can get all the names of the child controls with the [`keyof` operator](https://www.typescriptlang.org/docs/handbook/2/keyof-types.html).

```ts
type TheFormPath = keyof TheForm['controls'];
// which resolves to:
type TheFormPath = "name" | "email" | "address";
```

This is the "not nested" part of our union!

We can write it in a reusable way:

```ts
type FormControlPath<TControl extends AbstractControl> =
  TControl extends FormGroup | FormArray
    ? keyof TControl['controls']
    : never;

type TheFormPath = FormControlPath<TheForm>;
// which resolves to: "name" | "email" | "address"
```

With the "extends AbstractControl" we constrain the input of our mapping type to something derived from `AbstractControl`.
After that we distinguish between those kind of controls, which has children and those without children.
If we have child controls, we map to their names, otherwise we map to `never`, which is how you tell TypeScript to ignore this branch (or end the recursion, as we will see).

We could constrain the input to "extends FormGroup | FormArray", but we have to able to manage every control, so I made two steps at once here.
And we have to learn about ["conditional types"](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html) anyway.

### Adding Recursion

We can use the `keyof` operator to get the names of the child controls, but there's another, more complicated way.
This is like the moment in maths, where "you add something here and substract it over there again (the creative zero), so we can make better conclusions"...

Instead of mapping to the keys, we map to an object with the same keys as our "child controls object", but we type them with the name of its key.
And then we extract all the types of the keys (which are the names of the keys).
Yes, I'm totally aware of how this sounds...

```ts
type FormControlPath<TControl extends AbstractControl> =
  TControl extends FormGroup | FormArray
    ? {
        [K in keyof TControl['controls']]: K;
      }[keyof TControl['controls']]
    : never;
```

But with this in place we can investigate the type of the control behind the key: `TControl['controls'][K]`

- `TControl` is our input `FormGroup` or `FormArray`.
- `TControl['controls']` is the object with the child controls (that object is an array in case of `FormArray`, but that's equivalent to an object with numbers as keys).
- `TControl['controls'][K]` is the type of the child control with the name of the current key.

With some practice you get used to such kind of things, I promise!

Now we can add the recursion:

```ts
type FormControlPath<TControl extends AbstractControl> =
  TControl extends FormGroup | FormArray
    ? {
        [K in keyof TControl['controls']]:
          TControl['controls'][K] extends FormGroup | FormArray
            ? FormControlPath<TControl['controls'][K]>
            : K;
      }[keyof TControl['controls']]
    : never;

type TheFormPath = FormControlPath<TheForm>;
// which resolves to: "name" | "email" | "street" | "streetNo" | "city" | "zipCode" | "country"
```

We have now the names of all the `FormControl`s, but they are missing their prefix `address.` and the `address` itself is missing.

The latter is easy to fix, we have to add the name of the `FormGroup | FormArray` to the union of the names of its children.

```ts
type FormControlPath<TControl extends AbstractControl> =
  TControl extends FormGroup | FormArray
    ? {
        [K in keyof TControl['controls']]:
          TControl['controls'][K] extends FormGroup | FormArray
            ? K | FormControlPath<TControl['controls'][K]>
//            ~~~
            : K;
      }[keyof TControl['controls']]
    : never;

type TheFormPath = FormControlPath<TheForm>;
// which resolves to: "name" | "email" | "address" | "street" | "streetNo" | "city" | "zipCode" | "country"
```

Adding the prefix is not hard, but we have to take extra care.
The `FormControlPath<TControl>` return a union of key names.
With the use of [template literal types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html) we can map over them and add the current prefix.

```ts
`${K}.${FormControlPath<TControl['controls'][K]>}`
//   ^ here's the dot as the path separator
```

But we get one of the famous TypeScript error messages:

```
Type 'K' is not assignable to type 'string | number | bigint | boolean | null | undefined'.
  Type 'keyof TControl["controls"]' is not assignable to type 'string | number | bigint | boolean | null | undefined'.
    Type 'string | number | symbol' is not assignable to type 'string | number | bigint | boolean | null | undefined'.
      Type 'symbol' is not assignable to type 'string | number | bigint | boolean | null | undefined'.
```

(and a similar one for the second `${...}`)

What this means is, that TypeScript is not sure if `K` is of a type, which is allowed inside a template literal.
We have to constrain `K` to something, that is allowed.
Since we are dealing with forms, we are only interested in keys, which are `string`s or `number`s (remember: `FormArray`s).

We can add the constrain with an [intersection](https://www.typescriptlang.org/docs/handbook/2/objects.html#intersection-types):

```ts
`${K & (string | number)}.${FormControlPath<TControl['controls'][K]> & (string | number)}`
```

Looks weird? Yes.

```ts
type FormControlPath<TControl extends AbstractControl> =
  TControl extends FormGroup | FormArray
    ? {
        [K in keyof TControl['controls']]:
          TControl['controls'][K] extends FormGroup | FormArray
            ? 
              | K
              | `${K & (string | number)}.${FormControlPath<TControl['controls'][K]> & (string | number)}`
//                   ~~~~~~~~~~~~~~~~~~~                                             ~~~~~~~~~~~~~~~~~~~
            : K;
      }[keyof TControl['controls']]
    : never;

type TheFormPath = FormControlPath<TheForm>;
// which resolves to:
// | "name" | "email" | "address"
// | "address.street" | "address.streetNo" | "address.city" | "address.zipCode" | "address.country"
```

But it works! 🥳

### Don't forget `FormArray`!

Let's extend our example with a `FormArray`.
They are important and often ignored...

```ts
const fb: NonNullableFormBuilder = ...;
const form = fb.group({
  name: fb.control<string>(''),
  email: fb.control<string>(''),
  address: fb.group({
    street: fb.control<string>(''),
    streetNo: fb.control<string>(''),
    city: fb.control<string>(''),
    zipCode: fb.control<string>(''),
    country: fb.control<string>(''),
    lines: fb.array<string>([]), // <--
  }),
});

type TheFormPath = FormControlPath<typeof form>;
// which resolves to:
// | "name" | "email" | "address" | "address.street" | "address.streetNo" | "address.city" | "address.zipCode" | "address.country" | "address.lines"
// | "address.lines.at" | "address.lines.push" | "address.lines.length" | "address.lines.concat" | "address.lines.indexOf" | "address.lines.lastIndexOf"
// | "address.lines.slice" | "address.lines.includes" | "address.lines.toLocaleString" | "address.lines.toString" | "address.lines.pop"
// | "address.lines.join" | "address.lines.reverse" | "address.lines.shift" | "address.lines.sort" | ... 18 more ...
// | `address.lines.${number}`;
```

What happened?

We expected ``"address.lines" | `address.lines.${number}` ``.
All the other keys are the names of all the functions an array provides.
We don't want them in the result!

The "else" part of the inner condition just maps everything, which is not a `FormGroup | FormArray` to the name of the property.
This is, where we should further constrain our properties to those, that are actually derived from `AbstractControl`.
Everything else should be ignored.

```ts
type FormControlPath<TControl extends AbstractControl> =
  TControl extends FormGroup | FormArray
    ? {
        [K in keyof TControl['controls']]:
          TControl['controls'][K] extends FormGroup | FormArray
            ? 
              | K
              | `${K & (string | number)}.${FormControlPath<TControl['controls'][K]> & (string | number)}`
            : TControl['controls'][K] extends AbstractControl // <--
              ? K                                             // <--
              : never;                                        // <--
      }[keyof TControl['controls']]
    : never;

type TheFormPath = FormControlPath<TheForm>;
// which resolves to:
// | "name" | "email" | "address"
// | "address.street" | "address.streetNo" | "address.city"
// | "address.zipCode" | "address.country"
// | "address.lines" | `address.lines.${number}`
```

Done! 🥳 (now for real!)

With this "simple" mapping type we can improve our `debounceForm` function:

```ts
export const debounceForm = <TControl extends AbstractControl>(
  $form$: ValueSource<TControl>,
  debounceMs: number,
  ...debounceOn: FormControlPath<TControl>[]
//               ~~~~~~~~~~~~~~~~~~~~~~~~~
): Observable<FormValueOf<TControl> | null> => {
  // ...
};
```

Now, if someone (not me of course!) is trying to pass an invalid path to a nested control,
we will get an error at compile time - not at runtime!

I really prefer compiler errors over runtime errors...

For me TypeScript is like thin layer of unit tests, which are constantly executed, while I code.
I can not imagine building something larger than a simple example without it.

Understanding and being able to create "Mapping Types" is in my opinion a very important skill.

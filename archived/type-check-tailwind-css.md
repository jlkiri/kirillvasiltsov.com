---
title: "Make your CSS safer by type checking Tailwind CSS classes"
date: "2020-12-22"
spoiler: Make your CSS safer
language: en
---

[Tailwind CSS](https://tailwindcss.com/) is quickly gaining popularity among developers. I've noticed that it fits well in TypeScript projects. It turns out you can use the freshly released [Template Literal Types](https://devblogs.microsoft.com/typescript/announcing-typescript-4-1-beta/#template-literal-types) to type check your CSS!

## TL;DR

```typescript
type Tailwind<S> = S extends `${infer Class} ${infer Rest}`
  ? Class extends ValidClass
    ? `${Class} ${Tailwind<Rest>}`
    : never
  : S extends `${infer Class}`
  ? Class extends ValidClass
    ? S
    : never
  : never;
```

## A small recap of Template Literal Types

First of all let's see what these types are.

- For quite some time now, it's been possible to use literal types in TypeScript

```typescript
type MagicNumber = 42;
type MagicString = "magic";
```

- In JavaScript you can embed literals in template strings:

```typescript
const name = "John";
const age = 42;
const person = `${name}: ${age}`;
// "John: 42"
```

- Since 4.1, you can embed literal type **definitions** inside special literal string type **definitions** which are called Template Literal Types:

```typescript
type Name = "John";
type Age = 42;
type Person = `${Name}: ${Age}`;
// type Person = "John: 42"
```

It's OK to embed _unions_ of literal types too. In this case, although the Template Literal Type definition looks like it's one string, what you get is a union of strings, that reflects all possible combinations:

```typescript
type First = "A" | "B";
type Second = "A" | "B";
type Combined = `${First}${Second}`;
// ???
```

What does `Combined` look like? Behind the scenes, TypeScript calculates all possible combinations for you:

![table.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/397228/797f39a4-8c8b-425f-93bd-f5473ef211a8.png)

So the resulting type is `"AA" | "AB" | "BA" | "BB"`!

## Simple type checking of utility classes

Tailwind CSS comes with a lot of utility classes, but they are not random and follow a certain pattern. For example, all background color classes start with `bg` and end with luminance of `100` through `900`. The number of default colors at the moment of writing is around 10. For example: `bg-red-400`, `bg-purple-900` etc. So, you can type all possible background colors like this:

```typescript
type BgColor = 'bg-red-100' | 'bg-red-200' | 'bg-red-300' | ...... 'bg-purple-800' | 'bg-purple-900';
```

However, this is very unmanageable. If new luminance like `1000` or new color like `mauve` is added, you will need to add every combination to this huge union. That's exactly where template literal types come to rescue:

```typescript
type Colors = "red" | "purple" | "blue" | "green";
type Luminance = 100 | 200 | 300 | 400 | 500 | 600 | 700 | 800 | 900;
type BgColor = `bg-${Colors}-${Luminance}`;
// type BgColor = "bg-red-100" | "bg-red-200" | "bg-red-300" | "bg-red-400" | "bg-red-500" | "bg-red-600" | "bg-red-700" | "bg-red-800" | "bg-red-900" | "bg-purple-100" | "bg-purple-200" | "bg-purple-300" | ... 23 more ... | "bg-green-900"
```

We can do the same for [Space Between](https://tailwindcss.com/docs/space) classes, only this time also taking into account breakpoint variants.

```typescript
type Distance = 0.5 | 1 | 1.5 | 2 | 2.5 | 3 | 3.5 | 4 | 5 | 6 | 7 | 8 | 9 | 10;
type Breakpoints = "xs:" | "sm:" | "md:" | "lg:" | "xl:" | "";
type Space = `${Breakpoints}space-${"x" | "y"}-${Distance}`;
// type Space = "space-x-0.5" | "space-x-1" | "space-x-1.5" | "space-x-2" | "space-x-2.5" | "space-x-3" | "space-x-3.5" | "space-x-4" | "space-x-5" | "space-x-6" | "space-x-7" | "space-x-8" | ... 155 more ... | "xl:space-y-10"
```

## Type check arbitrary strings that contain Tailwind classes

It's cool that we can type check background colors or margins, but in real world we would be passing long arbitrary strings with classes like `bg-purple-900 space-x-2 xl:space-x-4`. How can we type check them as well? There is no regularity and order in these strings.

However, we do expect them to be a bunch of valid Tailwind classes that are separated by space. That's enough regularity. We can type check them with Template Literal types too! Let's implement the type step by step.

- First of all let's outline the new type. For now it's a simple conditional type:

```typescript
type Tailwind<S> = S extends ValidClass ? S : never;
```

Here, `ValidClass` is a union of all possible Tailwind classes. For the sake of this example, let's imagine that only valid classes are the `Space` and `BgColor` that we constructed above. So when we pass some string as a generic `S`, our type becomes the literal that we passed if it is a valid class, and `never` if it is not.

```typescript
type Good = Tailwind<"bg-red-400">;
type Bad = Tailwind<"bg-red-4030">;
// type Good = "bg-red-400"
// type Bad = never;
```

- Next, let's see how the type would be used in e.g. a function:

```typescript
function doSomethingWithTwClass<S>(cls: Tailwind<S>) {
  return cls;
}
```

If we pass the wrong class, it's properly detected and highlighted by TypeScript:

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/397228/13a7fb8c-233d-9932-f3a0-c6220e9dc4f6.png)

- Next, let's allow space. This is surprisingly the hardest part and feels like magic. As you may know, it is possible to use the `infer` keyword in TypeScript conditional types. This allows us to use the inferred type inside `true` expressions. This applies to template literal types as well. We can do something like this:

```typescript
type Tailwind<S> = S extends `${infer Class} ${infer Rest}` ? ...
```

By doing this we can infer the string before the space and the rest of the string right after the space.

```typescript
type Tailwind<S> = S extends `${infer Class} ${infer Rest}`
  ? Class extends ValidClass
    ? `${Class} ${Tailwind<Rest>}`
    : never
  : never;
```

If the inferred string is a valid Tailwind class we concatenate it and the type checking result of the rest of the string, which is done recursively. As soon as there is a wrong class, the type becomes `never`. There is one thing we forgot to consider though: sometimes the string has only one valid class, and no spaces, so we need to handle that as well:

```typescript
type Tailwind<S> = S extends `${infer Class} ${infer Rest}`
  ? Class extends ValidClass
    ? `${Class} ${Tailwind<Rest>}`
    : never
  : S extends `${infer Class}`
  ? Class extends ValidClass
    ? S
    : never
  : never;
```

And that's it! You can see that it works below:

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/397228/9495801b-1005-2fc6-311b-7feefe971840.png)

For those who want to play with it more, here's the [TS Playground](https://www.typescriptlang.org/play?#code/C4TwDgpgBAwg9gGzgJwM5QLxQOTIgE2ygB8cwBXZMBCI07AIwXNpJwHM8IA7bAbgBQoSFAAy5ALYBLbgENuAY2hYAjAAY1bAEwa2AZl2kALIagBWUwDZTAdlMAOUwE4Ng4dABC7eEmSYoAAYM7AC0ACQA3j4oqAC+4RHi0nKKELEBgkLg0AAiUqjA8kr+agB0ZmwqleXa2jWkevr1UEZsFaSWbDZs9mxOlWpu2VAeeLIA1mBwMsDoWNgAHqgAXHQ4qBKrbNgS+Fv0COz7OAsIx9j8WSIAymCyxVgBkaMQE1Mzcah3SgmLa9ggbDxSJ5ApFNIZARXaAANVkCCk+BgCFkqDmUFu92gpC80WQmXcUAAKrIpAgAO4yfAAHmuAD5-NcoBAFsAePh0E8IjIAGYQPzI1FxKCRXn8qAAJQgBXSAig8qgAH5YCi0czWez0HCEUjVag5QrDcquYK0bERRESWTKdwaVKCnTZYbDcsoNwIAA3fkG+Wupkstm2zmi7h8gV6p3O+XK03oAOaqDaxGxqA+qMK5XXNPpqCu91e5DZhV5z38zI88iKYBSODcKD4ODXOASCDAAAWMnYAHUpO2ieTY7S6QAKBQIFbE0kUqlDgCUUAiabwwEodbHqEEsShClrBSgDFk+H8DabLfbnZ7fYHeuHACJgiE8PgQiZNA+KFQaC-dMhb7PBDu3B7ge+BaMejbNq2HbcN2vZtv2sZ3gwIQftQEDfmof4AbuwD7oejRYCekHnjBl7wdeQp3lhAiAcBh6tIREFntBsFXoh96hE+ISOJoEhHF8WIhAsL7UbRuHsHAcBHoxp5QRecEITeHGPgQGFQAJPzCa0uzLBp6HCfYok4VAElSWBMnESxZGKZRylca+6nfPpIn-jRxmmfgBH1kxcmkQpFFoneelCSEKg1A+TAsCEOiYf+QA) link.

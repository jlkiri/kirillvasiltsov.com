---
title: Optional arguments in Rust
date: "2020-09-18"
spoiler: Two ways.
language: en
tags:
  - rust
---

If you come to Rust from languages like Javascript which have support for optional arguments in functions, you might find it inconvenient that there is no way to omit arguments in Rust. However, there are good ways to simulate optionality and I'm even starting to think that they make code more readable than simply omitting things here and there.

## Approach #1: two functions

This is the least creative approach, but absolutely legitimate. The idea is to create **two** functions with different number of arguments.

```rust
fn add_two(a: usize, b: usize) -> usize {
    a + b
}

fn add_three(a: usize, b: usize, c: usize) -> usize {
    a + b + c
}
```

The third argument here is "optional" in the sense that you have the option to not use the function and use `add_two` instead.

## Approach #2: Option<T>

`Option` is meant to represent something that might exist or might not exist. We can definitely use it to represent optionality in function arguments!

```rust
fn greet_name_optional(name: Option<String>) {
    println!("Hello, {}!", name.unwrap_or(String::from("world")))
}

greet_name_optional(None);
greet_name_optional(Some(String::from("Ferris")));

// Hello, world!
// Hello, Ferris!
```

`unwrap_or` is handy when you want to provide a "default" value if `None` is passed. The cons of this approach is that you still have to pass `None` to `greet_name_optional` when there is nothing else to pass and wrap existing values in `Some`.

## Approach #3: Enum

Just like `Option` is simply an `enum`, we can create our own enum that better represents the thing that the function expects.

```rust
enum Name {
    Empty,
    Is(String),
}
```

This also allows us to provide the default value by implementing the `Default` trait:

```rust
impl Default for Name {
    fn default() -> Self {
        Name::Is(String::from("world"))
    }
}
```

The function then becomes very intuitive and easy to read:

```rust
fn greet_name(name: Name) {
    match name {
        Name::Empty => println!("Hello, {}!", Name::default()),
        Name::Is(value) => println!("Hello, {}!", value),
    }
}

greet_name(Name::Empty);
greet_name(Name::Is(String::from("Ferris")));

// Hello, world!
// Hello, Ferris!
```

I personally prefer the third approach for its readability, but it requires some work, because you might need to implement other traits like `Display` too.

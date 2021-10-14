---
title: Parsing indentation with nom
date: "2021-10-14"
---

`nom` is an awesome parser combinator written in Rust and I believe it needs no introduction at this point. If you've never heard about it you wouldn't be reading this article anyway.

So, I needed to parse indendation for one of my projects ([Beemo](https://github.com/jlkiri/beemo) programming language) but couldn't find a convenient primitive in `nom`. One reason is that `nom` parsers do not really have access to any kind of state and it's not easy to make them do so. However it turns out it is not very hard if you wrap your parsers properly.

First of all, it's important to clarify the goal: we want indentation to be visible to our own parser and basically function as invisible braces, which means we should generate both INDENT and DEDENT tokens. The simplest way to do so is to keep track of current indentation _level_ and generate an INDENT token when it goes up and a DEDENT token when it goes down.

Long story short, here's the parser:

```rust
fn indentation<'a>(input: &'a str, counter: &mut IndentationCounter) -> Result<'a, Vec<Token>> {
    let (rest, tabs) = many0(tab)(input)?; // Count tabs in the current line.
    let mut indent_tokens = vec![];
    let indent_level = tabs.len() as isize;
    if indent_level < counter.current {
        for _ in 0..counter.current - indent_level {
            indent_tokens.push(Token::Dedent); // Dedent if fewer tabs than in the previous line.
        }
    } else if indent_level > counter.current {
        for _ in 0..indent_level - counter.current {
            indent_tokens.push(Token::Indent); // Indent if more tabs than in the previous line.
        }
    }
    counter.current = indent_level; // Update current indentation level
    Ok((rest, indent_tokens))
}
```

The counter is just a very simple struct:

```rust
#[derive(Debug)]
struct IndentationCounter {
    current: isize,
}
```

Here's how you would use it in real code.

```rust
fn scan(source: &str) -> Vec<Token> {
    let mut c = IndentationCounter { current: 0 };
    let (_, tokens) = scan_lines(&source, &mut c).expect("Failed to scan.");
    tokens
}

fn scan_lines<'a>(source: &'a str, counter: &mut IndentationCounter) -> Result<'a, Vec<Token>> {
    let indent = |i| indentation(i, counter); // Valid nom parser.
    let (rest, result) = /* Do usual nom stuff */
    Ok((rest, result))
}
```

I want to emphasize that `|i| indentation(i, counter)` is a valid `nom` parser as long as our `indentation` function returns the expected output. So, if we parse a text file that looks like this:

```
a
    b
        c
    d
e
```

We get the following output:

```
/*
    Var(
        "a",
    ),
    Indent,
    Var(
        "b",
    ),
    Indent,
    Var(
        "c",
    ),
    Dedent,
    Var(
        "d",
    ),
    Dedent,
    Var(
        "e",
    ),
]
 */
```

Note that this examples assumes that nothing in your input can be multiline. I'm not sure how this approach would translate and maybe all you'll need to do is to handle a couple of edge cases. Let me know if there are other better ways to parse indentation with `nom`!

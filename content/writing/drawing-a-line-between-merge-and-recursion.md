---
title: Drawing a line between Merge and recursion
date: "2019-12-01"
spoiler: Merge is not really recursive.
language: en
tags:
  - linguistics
  - post
---

This post was motivated by two recent papers: one [by Martins & Boeckx](https://journals.plos.org/plosbiology/article?id=10.1371/journal.pbio.3000389) and another [by Berwick & Chomsky](https://journals.plos.org/plosbiology/article?id=10.1371/journal.pbio.3000539).

WARNING: This post contains some examples of code written in a programming language, but the main point of this post does not depend on understanding them. Feel free to skip it if you are uncomfortable with code.

Basically, both papers focus on the question whether such thing as half-Merge is possible. Martin & Boeckx (M&B) suggest that External Merge and Internal Merge could be distinct steps in the evolution of human language capacity. The former builds "nested dependencies", and the latter allows "crossed dependencies". M&B argue that IM (the dependency it creates) is more computationally complex and so believe that out of two steps it is the second one. On the other hand, Berwick & Chomsky (B&C) point out that IM and EM can be seen as mere subcases of a single operation Merge. They point out that in fact, IM and EM as distinct operations would be more complex because they place restrictions (such as "must be distinct/subset of") on things merged. And even if they were distinct, as B&C notice, EM would be more computationally complex than IM, because it requires massive search through the lexicon.

But one thing in both papers intrigued me more than the central argument. Interestingly, M&B seem to agree that while half-Merge is possible, half-recursion is not. What does that mean?

## Not "Merge IS recursion", but "Merge AND recursion"

That means that M&B subscribe, accidentally or not, to the view that Merge itself is not recursive. In fact, even B&C imply the same in their response to M&B in the part where they counter M&B's argument about complexity of nested and crossed dependencies:

> Merge-based systems do not even appear in the [Chomsky] hierarchy, and anything concluded from the study of the Chomsky hierarchy is totally irrelevant to the evolution of Merge-based systems

In other words, what was once used to explain potential infinity of syntactic structures, that is, recursive context-free grammars, is now irrelevant because Merge does not even produce strings (with linear order), and Chomsky hierarchy was not meant to be applied to nested sets. This is what confused me first, but then I realized that all this time I've been taking for granted the vague concept of recursion in human language.

In literature, Merge is sometimes said to be recursive **because** it can be applied to its own output. While that is an important property that indeed makes nesting (and potentially infinite nesting) possible, this is **not** what makes it recursive. In fact, if you look at the definitions of recursion in computer science or mathematics literature, no definition says that a function (and Merge is a function) is recursive iff it can be applied to its own output.

Roughly speaking, a function is recursive when it is defined in terms of itself.

It is very easy to see the difference: the extremely simple _identity_ function (`λx.x` or `id(x) = x`) can be applied to its own output, but it is not recursive (= not defined in terms of itself):

```javascript
id(1) = 1
id(id(1)) = 1
id(id(id(1))) = 1
...
```

A good example of a truly recursive function would be a function that calculates the nth power of a number by referring itself:

```javascript
pow(num, 0) = 1;
pow(num, n) = num * pow(num, n - 1);
```

Note that the base case is required in definitions of this sort to be correct. Above it means that the 0th power of any number is 1. For example, the 3rd power of 3 would be calculated this way:

```javascript
pow(3, 3) = 3 * (3 * (3 * 1)) = 27;
```

Importantly, `pow` **cannot** be applied to its own output alone. It outputs one number but requires 2 arguments! However, the repeated multiplication as a result of recursive application of `pow` above is what seems to be meant by recursion in linguistic literature. Merge can obviously be repeatedly applied too:

```javascript
merge(a, merge(b, merge(c, merge(d, e)))) = {a, {b, {c, {d, e}}}};
```

But Merge here is more like multiplication in the example above: it is a **mean** to produce the output, once or many times, but it is **not defined in terms of itself**. Strictly speaking, then, **Merge alone is not enough to generate potentially infinitely nested structures**. There must be another, recursive function or whatever "routine" similar to `pow` above, that keeps applying Merge again and again until the base case is met. There must be both recursion AND Merge to achieve what we observe in human syntax.

That is, of course, if we want to be really pedantic. But since the supposedly unique ability of humans to Merge again and again is brought up so much in discussions about the evolution of language, I think we do need to be pedantic.

## Recursive data structures by non-recursive means

On the other hand, the _structure_ that can be produced by repeated application of Merge _is_ formally recursive. Just like a tree can be composed of leaves or smaller trees, a structure produced by Merge can either contain two "atoms" (e.g. lexical items) or an atom and a set that itself contains two atoms (or an atom and another set) etc. In a programming language which allows definition of recursive data structures (e.g. Haskell) it can be defined like this:

```haskell
data SyntacticObject a = Atom a | Set a (SyntacticObject a)
-- where a is a type of an atom
```

However it is also important to note that procedures that are capable of building such a structure need not be recursive. If the way our brain computes is similar to how computers do their work (= Turing machines) then a syntactic structure may as well be built by some complex sequence of commands, that is not recursive at all and even uses a hypothetical `replace` function that I just came up with:

`1. Build {a, ∅}`  
`2. Build {b, ∅}`  
`3. Build {c, d}`  
`4. Replace ∅ in {b, ∅} with output of 3`  
`5. Replace ∅ in {a, ∅} with output of 4`

The above yields `{a, {b, {c, d}}}`. How do we know that syntactic structures are not built this way? We don't. All we really know from years of syntactic research is that they are likely to have and be required to have a form defined above. And Merge (with whatever repeatedly applies it) is the most succinct way to define an operation which might build these structures. If, however, the real algorithm is like the one above, then it is not the ability to recurse that did magic in homo sapiens, but rather increased memory capacity, since each individual output of Merge must be stored somewhere. And it is the algorithm that matters when we talk about evolution.

## What recursive Merge might look like

But suppose that recursion indeed plays a big role in human language. I see then two possibilities:

1. Some general ability to recurse is applied to Merge
2. Merge itself is recursive (and needs another definition)

In the first case, we need to look for some more general ability to apply any function a potentially unbounded number of times. As far as I know, recursion can actually be seen as a _self-referential composition_ of functions. That is, there is nothing extraordinary about recursion, just like there is nothing extraordinary about function composition. When we compose two functions `f` and `g` we produce another function `h` such that when it is applied to `x`, `x` first becomes an input to `f` and then its output becomes an input to `g`. If `f` and `g` are the same function then we get the same result by their combination as if `f` itself was defined recursively. And combination itself is nothing more than yet another function that produces this behaviour when applied to two functions. We can come up with a function `applyTimes` that would keep combining another function `f` with itself an arbitrary number of times. For no specific reason the idea can be expressed in Haskell like this:

```haskell
applyTimes 0 f arg = arg
applyTimes n f arg = f . applyTimes (n-1) f $ arg
-- where (.) is a function that combines two of its arguments
```

Thus if we want to combine some function `f` 4 times we basically get this result:

```haskell
applyTimes 4 f arg = f . f . f . f $ arg
-- or in a more familiar notation f(f(f(f(arg))))
```

Put simply, Merge does not need to be recursive if something else can do the multiple application of Merge instead.

In the second case, Merge needs a new definition if we want it to be truly recursive. I couldn't come up with a good one, but I think that capital MERGE is a better fit (if I understood it correctly). Mainly because it accepts only one argument, which is a collection of e.g. lexical items. (It is not impossible to compose functions that accept more than one argument but that itself requires a process that combines them first, which is weird.) The process would be something like this: take a collection of lexical items, combine two of them into a set (by some other function like `combine`) and pass the modified collection to itself, until the only item in the collection left is a nested set (= base case).

1. `MERGE([a, b, c, d])`
2. `MERGE([{a, b}, c, d])`
3. `MERGE([{c, {a, b}}, d])`
4. `MERGE([{d, {c, {a, b}}}])`

Note again that even when a function is recursive, in the physical world (of classical computers, at least) it needs to know how many times it should be applied, because otherwise it will apply infinitely, which is not possible with limited memory. The number of times MERGE should apply can probably be easily derived from the size of the collection that needs to be merged.

## What matters

To summarize, Merge as currently defined is definitely not recursive. However, whether it is really recursive or not should not matter much for syntactic-theoretical purposes as the focus is on the form of the structure that a single Merge application generates. On the other hand, it does seem to matter in evolang research, because the ability to recurse might be independent from Merge and might be found elsewhere. Unlike with computers, it is not known whether iteration and recursion are implemented differently in the brain. But physically both require memory to be possible, so depending on a metaphor one is using, it might make more sense to focus on gradual increase of "memory capacity". As far as I know, Progovac actively explores the possibility that protolanguage could consist of single applications of Merge, which basically yields a structure with only two words.

The actual algorithms and their implementation have to do with what is considered as language use, as opposed to _competence_. I personally still find this dichotomy useful once we figure out our precise definitions as I tried to do above. I do not think that focusing to much on difference between EM and IM is useful since it is a question about competence and not language use (including performance), precisely what seems to me to be a good target for evolang research.

That said, I do find interesting the theoretical possibility that Merge and recursion are different things (and it simply makes more sense). Especially if recursion can be seen more abstractly as self-referential composition of two actions, it should be interesting to look for this kind of behaviour in other animals.

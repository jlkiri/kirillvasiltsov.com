---
title: Some thoughts on Merge
date: "2019-09-22"
spoiler: Trying to be really picky about Merge.
language: en
tags:
  - linguistics
  - post
---

The idea of Merge as a fundamental computational operation of human language has been all the rage since 1990s in generative linguitics.

However I have long felt that the original definition of Merge has largely served its purpose. A lot of things (like movement) in human language suddenly started to make sense when we first analyzed them as workings of Merge. Now, I personally struggle to find new phenomena which Merge would explain as brilliantly as it did movement (at least not in a far-fetched way).

A few days ago I [tried](https://twitter.com/virtualkirill/status/1172450475976712192) [to be picky](https://twitter.com/virtualkirill/status/1172460087387934720) about Merge on Twitter, and realized that it might be better to write a post about it.

### Sentences, rivers and lungs

Hierarchical structure is observed in many aspects of human language. Syllables, words, and sentences all have hierarchical structures. In the first case we see nuclei and codas combine into rhymes. In the second case we see roots and affixes combine into words. In the third case we see verbs and nouns combine into verb phrases etc.

All this makes the idea that "all you need is Merge" tempting. Whatever hierarchical structure you may find in human language -- Merge could explain it!

The problem is, we never see codas combine with nouns to form a larger, linguistically meaningful object, or sentences which structure is defined only by recursive combinations of nuclei. It is as if these entities live in different worlds and are not compatible with each other. If that is so, maybe Merge is not all we need after all.

In this [article about Merge](https://nautil.us/issue/76/language/this-simple-structure-unites-all-human-languages) David Adger nicely shows how hierachical structure (by Merge) is not a big surprise given that self-similarity is found all over the place in nature -- lungs branch, rivers branch, etc.

One interesting way to interpret the article is this: if self-similarity just exists, in lungs, in rivers, independently, then why bother so much about Merge in the first place? The article focuses on Merge as if it is special, by arguing that it is _not special_. After all, no one is trying to explain branching of lungs or rivers by some computational operation like Merge. And certainly no one is trying to find what is common between lungs and rivers in particular. It is obvious that branching rivers cannot become lungs or vice versa. What if exactly because self-similarity is so common, there is no problem with pieces of syllables being incompatible with pieces of sentences?

The reason Merge feels special is because we are talking about structure in syllables, words and sentences -- all of which are building blocks of human language. So it is tempting to think that Merge somehow made all this possible and that without Merge we would have neither syllables, nor words or sentences. This, however, is an empirical question.

Of course, the problem of "compatibility" only arises if one thinks of Merge as an unconstrained computational operation that just combines everything. In fact, Merge, as originally defined by Chomsky, combines and produces **syntactic objects**, which are basically lexical items or combinations of lexical items. Nothing is said about whether Merge is actually capable of building syllable structures or morphological structures. And as I said above, I believe there is nothing wrong about not stretching the definition of Merge to other structures. So we could stop right here and it would be fine, but I simply found it interesing to try to find a more rigorous definition. This is what I do in the next section.

### What can Merge merge?

Let us take the simplest definition of Merge possible.

```javascript
merge(x, y) = { x, y };
```

This reads: "for _any_ `x` and `y`, the product of their Merge is a set `{x, y}`". As I mentioned above, this cannot be true because not any `x` and `y` are of the same **type**. The original definition of Merge quite explicitly constrains the types of `x` and `y` to lexical items. To make this explicit in our formal definition we would write something like this:

```haskell
(LI x, LI y) => merge(x,y) = {x, y}
-- where LI stands for lexical item
```

It is interesting to see what this would mean in the world of programming languages. The first definition of Merge makes it _parametrically polymorphic_. This is a fancy way of saying "for _any_ `x` and `y`, the product of their Merge is a set `{x, y}`". The second definition is an example of _ad-hoc polymorphism_. Basically, it means that `merge` is defined explicitly only on lexical items.

To put it differently, in the second case, we can say that there is an abstract class of Mergeable things that is defined by their _ability_ to merge, but does not specify _how_ they should merge. It becomes a responsibility of every concrete class to specify (or not) what it means to merge them. And by concrete classes I mean e.g. syllable parts or sentence parts.

If we look at it from this angle, then maybe Merge _is_ special but not as one all-mighty operation. Instead, Merge is a behaviour that concrete things can implement in its own way (and even produce similar structures, but of different types). Here, by structures I mean structures in mathematical sense, and not data structures like trees.

For example, it might even turn out that other parts of human cognition implement this Mergeable class behaviour, like numbers (which form a [monoid](https://en.wikipedia.org/wiki/Monoid)), but also have additional behaviour not observed in any aspect of human language. We can imagine that a class of natural numbers (in which we likely "think" when we do arithmetics) implements the Mergeable behaviour in the following way:

```javascript
1 + 2 = 3
-- or
merge({.}, {., .}) = {., ., .}
```

Now, it is usually believed that output of Merge in syntax can itself be a legitimate input to Merge (hence recursion). Mathematically, this means that e.g. lexical items _at least_ form a commutative [magma](<https://en.wikipedia.org/wiki/Magma_(algebra)>) under Merge. But monoids (natural numbers under addition) are also magmas, with additional properties of associativity and identity! It is in this sense that we can say lexical items and natural numbers have similar structures. Independent implementations of Mergeable behaviour do not necessarily need to produce similar _data_ structures -- numbers do not need to be tree-like.

I believe that capital MERGE is an attempt to explicitly define what we might call a magma of lexical items. Chomsky uses a fancy word "workspace" which really is a set in mathematical sense and may be misleading. This is a good idea, because we can ask questions like "Why magma? Why not semigroup?". [This tweet](https://twitter.com/davidadger/status/1172465989201608704) made me want to investigate more, and I even feel like we have already found some evidence of associativity in syntax, although it was not explicitly said so. A good example is what Hornstein and Uriagereka called "reprojections". Basically, if we take quantifiers like _most_ to be functions that take two arguments then the following sentence:

(1) Most people love children.

must have two structures at the same time: one in which _most_ has IP _love children_ as its specifier and one in which _most children_ itself is a specifier of IP _love children_. In the case of reprojection this is supposed to somehow magically happen during derivation.

I think that it may be useful to focus on how we think these two structures are related. From the meaning point of view we would like them to be identical. This would imply that at least in this particular case:

```javascript
{{x, y}, z} = {x, {y, z}}
-- or
merge(merge(x,y), z) = merge(x, merge(y, z))
```

where `x` and `z` stand for two different "maximal projections" and `y` stands for an "intermediate projection". It is, of course, completely unclear what it means for projections (or labels) to merge. But as a pattern, it shows associativity, which would mean that syntactic objects under Merge are a semigroup.

At this point, I do not see why it should be important whether we are dealing with a magma, semigroup or monoid or whatever in syntax, but I expect that we might benefit from thinking in more abstract terms than data structures like trees.

Finally, I believe that thinking of merge as an abstract behaviour which concrete classes may implement allows us to think about cognitive activities independently. In the case of one all-mighty Merge we cannot have a living organism that produces syllables (or harmonious sounds) but does not produce words or sentences. In the case of many independent merges, there are a plenty of possible combinations of abilities.

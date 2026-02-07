---
layout: default
title: Branded Types in Rust
summary: Branded types are a Rust design pattern that allows you to associate a bunch of values on the type level.
         This can allow you to statically verify some properties that would normally require a runtime check.
---

# Branded Types in Rust

[GhostCell](https://plv.mpi-sws.org/rustbelt/ghostcell/) research paper describes a zero-overhead alternative to a `RefCell` or a `Mutex`.
The application is in the implementation of data structures with internal sharing (e.g., graphs), something that people sometimes struggle with when working within the bounds of Rust's borrow checker.

I'm not going to go into detail here (better to read the paper itself), but the gist is that the shared value is wrapped inside a `GhostCell` that allows you to borrow its contents without any dynamic checks.
Instead, the "check" is performed statically, on the type level: there is a separate `GhostToken` value that acts like a key.
To borrow from a `GhostCell`, you need `&GhostToken` or a `&mut GhostToken` if you're borrowing mutably.
The token itself is a zero-size phantom value that you can pass around or store inside data structures to control which parts of the code have access to a whole family of cells.

To prevent using a token to unlock a cell that doesn't belong to it, this approach needs some kind of type-level association between cells and tokens.
This is accomplished using a slightly hacky "design pattern" that uses Rust lifetimes in an unconventional way.
Both `GhostCell` and `GhostToken` structs have a lifetime parameter `'id` that is generated in a way that makes it fully unique.
This lifetime is not associated with the duration of any particular borrow and is only used to match, on a type level, a token with the cells that are opened by it.
This means that, for a specific `'id`, `GhostToken<'id>` is a guaranteed singleton, i.e. there is exactly one value of that type.

## The Mechanism

The paper calls this mechanism a "branded type" and credits [You Can't Spell Trust Without Rust](https://faultlore.com/blah/papers/thesis.pdf) (Chapter 6.4) with its invention, or at least its introduction in Rust.
They also mention Haskell's `ST` monad and other functional programming ideas from the 90s.
None of the other references use the term "branded types" and there is no relation to the [branded](https://docs.rs/branded/latest/branded/) crate, so details and other uses can be hard to search for.

The main disadvantage of this pattern is in the process of creating those unique lifetime parameters.
The only way to construct a new `GhostToken` is not with a function that returns one, but with a higher-order function that you pass a closure to:

```rust
fn new<R>(f: impl for<'id> FnOnce(GhostToken<'id>) -> R) -> R
```

This exploits the fact that, for a universally-quantified parameter `'id`, the lifetime has to be assumed unique in the body of the passed function.
The `R` in this signature only serves to pass the results from your closure to outside and it can never contain any values that depend on `'id`.

The function `new()` is *rank-2 polymorphic*, i.e. its argument itself is a function that has type or lifetime parameters, and requires a type system expressive enough.
Rust only supports a restricted form of rank-2 polymorphism in the form of [Higher-Rank Trait Bounds](https://doc.rust-lang.org/nomicon/hrtb.html).
The clause `for<'id>` is only allowed as long as it is applied to lifetimes and not to type parameters.

If you need to create several of those tokens at the same time, it would be possible to have a more ergonomic API with multi-argument variants of `new()`.
This will reduce, but not completely eliminate, these nested closures.
If your use of the data structure is not well-scoped, the only option is to propagate the higher-order API to your functions.
This would essentially force you to structure a subset of your code in a [continuation-passing style](https://en.wikipedia.org/wiki/Continuation-passing_style).

## Other uses

Section 2.2 of the [GhostCell paper](https://plv.mpi-sws.org/rustbelt/ghostcell/) introduces the "branded type" concept with an example alternative API for vectors.
Each vector is parameterized with its own unique `'id` that is shared with instances of `BrandedIndex<'id>` that wrap a pre-checked index.
This kind of index can be created by pushing a new element, or by verifying an integer index against the vector's length once.
Retrieving elements can now be done safely without a dynamic length check.
The reason this needs branded types is to prevent using a pre-checked index from one vector with another one.

This is a good pedagogical example but, because of the construction mechanism disadvantage described above, I don't think I'd like to use vectors like these in ordinary code.
One place where the awkward API is not that big of a deal is when using branded types with connection- or session-like values.
Those are often well-scoped and there's a benefit from having static checks that prevent you from mixing up values belonging to different connections.

## Connections to functional programming

The authors of the paper mention that this mechanism is inspired by functional programming classics like the [Haskell ST monad](https://wiki.haskell.org/Monad/ST).
`ST` is a generalization of the `IO` monad and allows you to create a family of `STRef` handle-like values that you can use to embed stateful computation inside a pure function.
This is accomplished via a function with a signature very similar to that of `GhostToken::new()`:

```haskell
runST :: (forall s. ST s a) -> a
```

Haskell doesn't have lifetimes but this `s` parameter (called a "thread") is never a real type with values.
It is only used to index a family of `STRef` references and ensure that they can only be accessed within the `ST` monad indexed by the same `s`.
It's a similar solution to a similar type-level value association problem.

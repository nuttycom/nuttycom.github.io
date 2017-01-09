---
title: What things are, and what they mean.
---

This is a continuation of [this
discussion](https://twitter.com/raichoo/status/622002344390225920).  I'm
writing this because I need a bit more space to lay out my thoughts more
clearly.

The discussion around "typeclass coherence" is one that seems to divide even
people who generally agree on a lot of things about programming - that static
type systems are valuable, that modularity is desirable, and that being able to
reason about software in terms of composability and laws is necessary if we
ever want to be able to stop rewriting the entire ecosystem every few years,
just because it has accrued too much cruft. 

I'm going to make an analogy here to the IO type in Haskell before I dive into
my position on typeclass coherence. Initially, IO was devised as a mechanism -
a hack, really - to make it possible (or at lease easier) to reason about
effects in the presence of pervasive laziness. The IO type does this admirably,
but the way that it does it is that it gives us equational reasoning in the
presence of effects.  Now, there are good arguments that in some way, the
reasoning capability we gain is somewhat trivial, in that we've simply banished
the problem to an external system. Nonetheless, it's very clear that in terms
of the everyday process of writing software, being able to refactor by treating
effectful computations as values and being able to move them around freely is
very useful, as is the ability to use types to help us track and distinguisg
effectful computations as being separate from pure ones. We have discovered
that IO (and narrower effect types, encoded as free monads) have the most
meaning in terms of the feature - referential transparency - that they provide.

## Typeclass Coherence

Similarly, I think that something that is missing from the discussion is a
distinction between what a typeclass *is* (or perhaps, *how it came to be*) - a
hack to allow overloading in the context of Hindley-Milner type inference - and
the way that we have largely come to use typeclasses - and the meaning that
we've discovered in the process.

I see the term "typeclass" as short for "typeclass(ification)." When we write a
typeclass instance, we are classifying that type - but, due to what I kind of
regard as a historical accident - we do so using terminology that I regard as a
bit unfortunate, because of the way that it colors our perception of what we're
doing. I'm speaking here about the 'instance' keyword.

I think that 'classify' captures much better the sense of what we're doing when
we define the implementation of a typeclass for a particular type.  We are
(ideally) stating that the that type is usable in a small set of functions that
relate to one another by a set of laws. The fact that this can be though of as
implicit dictionary passing, and the fact that implementing a typeclass for a
type *looks* like overloading, is somewhat irrelevant. 

When we define, for example the implementation of the Monad typeclass for
Maybe, we make values of type `Maybe a` usable by the `(>>=)` function, and
make it possible for `return` to lift values of type `a` into values of type
`Maybe a`.  Notice that you can't call the `Mabye.(>>=)` function explicitly -
this is because the `(>>=)` defined inside of the `instance Monad Maybe
where...` block isn't actually a function in its own right! It is simply a
statement of how values of type `Maybe a` behave in the context of
`Monad.(>>=)`, which *is* a first-class, polymorphic function with a restricted
set of possible inputs.

A practice that I have adopted in my Haskell code that helps clarify this
distinction is that I never write the implementation of any typeclass function
for a type within the body of the 'instance' block. For example, instead 
of this:

```haskell
instance Monad Maybe where
  (Just a) >>= f = f a
  Nothing >>= f = Nothing

  return a = Just a
```

I consider it better to write this:

```haskell
bindMaybe (Just a) f = f a
bindMaybe Nothing = Nothing

pureMaybe a = Just a 

instance Monad Maybe where
  (>>=) = bindMaybe
  return = pureMaybe
```

This makes it clearer that we're defining what `(>>=)` and `return` mean in the
context of the `Maybe` type, and not just creating another implementation of
some functions named `(>>=)` and `return`.

Something important to note here is that the in the definition of the `Monad`
typeclass, all of the available type variables are abstract. We have no
additional structure that we can use to inspect obedience to the monad laws -
we are forced to to rely entirely on parametricity (and the unenforced monad
laws, which we at present must simply trust will be obeyed by the implementer)
when reasoning about the behavior of any application of `(>>=)` or `return`.

Compare this now to my favorite whipping-child typeclasses, `FromJSON` and
`ToJSON`.  These "typeclasses" (though not worthy of the name) expose the
`Value` type directly in their functions, and consequently destroy our ability
to reason parametrically. Even if we combined these typeclasses to create a new
`SerializeJSON` class, we'd still be simply using the typeclass mechanism to
allow us to overload a bunch of function names, and we're in fact limited by
the fact that we can't easily refer to the `MyType.parseJSON` function directly
- and if we want to refer just to that capability for a type, we're forced into
capturing the `FromJSON` typeclass instance existentially, which is widely (and
appropriately) considered a bad idea.

In summary, I think that we need to separate our ideas about overloading and
implicity dictionary passing from how we make it possible to use values of
newly defined types in the contest of globally-defined polymorphic functions.
Let's use typeclasses for the latter, and as for the former, for now I'm just
going to keep passing regular old functions or explicit dictionaries. Using
typeclasses in all the places we want to use overloading just muddies the
waters too much.

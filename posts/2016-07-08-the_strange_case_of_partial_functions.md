---
title: The Strange Case of Partial Functions
---

This is an extended response to the twitter discussion that arose from [this
tweet](https://twitter.com/hseeberger/status/751269895808032768).

There is a problem with PartialFunction in Scala, and I don't mean the fact
that, in the current standard library, PartialFunction is a subtype of
Function1. As I'll try to illustrate, the strange inheritance relationship is
the least of its problems. What I want to look at is what PartialFunction means
in terms of API design, and what we can learn from this with respect to what
the relationship between PartialFunction and Function1 might usefully look like
in a future version of Scala.

To begin with, let's contrast `PartialFunction[A, B]` with its semantic rival,
`A => Option[B]`. The most frequent justification I've heard for preferring the
former in some cases is that the result value is not boxed by an Option
instance, and that avoiding this boxing saves on memory and allocation overhead. Let's
call this the allocation argument.

So, let's think about what this means in terms of API design. Suppose that I
define a method taking a partial function as an argument:

```scala
def foo(f: PartialFunction[String, Int]) = ???
```

This signature gives us a little bit of information: it essentially promises to
the caller that I will, at some point in the evaluation of foo, call
`f.isDefinedAt(someString)`, and, given a positive result, that I will then
call `f(someString)`. Fair enough. There's something else, though; as the
author of this method, I had the option to define it as:

```scala
def foo(f: String => Option[Int]) = ???
```

In general in scala, we prefer to work with total functions wherever possible,
so by choosing *not* to define the argument to `foo` as returning `Option[Int]`,

I am making an explicit statement: that the combination of the overhead of
invoking `isDefinedAt` and then evaluating the function itself at the same
input *must be* less resource-intensive than evaluating the test once (during
the evaluation of the function) and then boxing the result. This is a
substantial requirement to impose upon the caller! It means that if
`isDefinedAt` performs a single allocation on the heap (say of a closure,
incredibly common in most Scala code) then we've probably already lost ground
relative to the `String => Option[Int]` version. Now, of course, there are a
few functions where `isDefinedAt` is pretty trivial to check; division testing
for a zero denominator, for example, but even this probably isn't as cheap as
one would hope - you have an (additional) megamorphic dispatch site at
`isDefinedAt`.

All of this is to say that defining an API in terms of `PartialFunction` places
some substantial burdens upon the caller to know and understand the performance
of the function they're passing to you, and that the set of functions where the
`isDefinedAt` check has lower overhead than boxing the result in Option is
likely quite small. This is made worse in present-day Scala, where if you 
wish to pass a total function to a method taking a PartialFunction argument,
you're going to be introducing a performance penalty for the indirection 
introduced by wrapping with `PartialFunction.apply`. 

Also implicit in the choice to use `PartialFunction` rather than `A => Option[B]`
is that the argument function will be called enough times that the performance
difference will actually matter. This is bad too: depending upon how
much of its domain the argument function is defined over, it may actually
be cheaper to catch any exceptions thrown by its unsafe invocation than it is to
check `isDefinedAt` for each input! So, this places a demand on the caller that
the the valid domain of the argument function is relatively sparse, or at 
least that it's frequently expected to be called with inputs for which it
is not defined.

Given all these constraints placed upon the caller, it's very very rare that
we actually want to define a function that takes a PartialFunction as an argument.
So rare that the presence of PartialFunction in the standard library itself
is a pretty questionable decision - it's a pretty innocent looking little 
trait, with a *lot* of subtle implications, all tied to performance, which we
know is a complex subject and which is something we don't want to optimize
for prematurely to begin with! 

If we consider instead APIs that return (rather than accept) PartialFunction
instances, we're hardly in better shape. Non-total pattern match literals
can generate partial functions automatically, which can make for *slightly*
cleaner DSLs. However, given the fact that very few APIs should be written
to consume those PartialFunctions, who is going to be around to consume them?

Overall, `PartialFunction` is tricky to deal with, frequently requires
ascription, and fundamentally doesn't pull its weight, and probably shouldn't
be included in a future version of Scala at all.

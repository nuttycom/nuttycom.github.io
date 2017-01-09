---
title: F-bounded Type Polymorphism Considered Tricky
---

This post is intended to address a question that I seem to see come up every
few months on the Scala mailing list or the #scala IRC channel on freenode. The
question is often posed in terms of the phrase "recursive type" and usually
involves someone having some difficulty or other (and *many* arise) with a
construction like this:

```scala
trait Account[T <: Account[T]] {
  def addFunds(amount: BigDecimal): T
}

class BrokerageAccount(total: BigDecimal) extends Account[BrokerageAccount] {
  def addFunds(amount: BigDecimal) = new BrokerageAccount(total + amount)
}

class SavingsAccount(total: BigDecimal) extends Account[SavingsAccount] {
  def addFunds(amount: BigDecimal) = new SavingsAccount(total + amount)
}
```

This sort of self-referential type constraint is known formally as (F-bounded type
polymorphism)[http://www.cs.utexas.edu/~wcook/papers/FBound89/CookFBound89.pdf]
and is usually attempted when someone is trying to solve a common problem of
abstraction in object-oriented languages; how to define a polymorphic function
that, though defined in terms of a supertype, will when passed a value of some
subtype will always return a value of the same subtype as its argument.

It's an interesting construct, but there are some subtleties that can trip up
the unwary user, and which make naive or incautious use of the pattern
problematic. The first issue that must be dealt with is how to properly refer
to the abstract supertype, rather than some specific subtype. Let's begin with
a simple example; let's say that we need a function that, given an account,
will apply a transaction fee for adding funds below a certain threshold.

```scala
object Account {
  val feePercentage = BigDecimal("0.02")
  val feeThreshold = BigDecimal("10000.00")

  def deposit[T <: Account[T]](amount: BigDecimal, account: T): T = {
    if (amount < feeThreshold) account.addFunds(amount - (amount * feePercentage))
    else account.addFunds(amount)
  }
}
```

This is straightforward; the type bound is enforced via polymorphism at the
call site. You'll notice that the type ascribed to the `account` argument is `T`,
and not `Account[T]` - the bound on `T` gives us all the constraints that we want.
This does what we want it to do when we're talking about working with one
account at a time.  But, what if we want instead to perform some action with a
collection of accounts of varying types; suppose we need a method to debit all
of a customer's accounts for a maintenance fee? We can expect our type bounds
to hold, but things begin to get a little complicated; we're forced to use
bounded existential types.  Here is the correct way to do so:

```scala
object Account {
  def debitAll(amount: BigDecimal, accounts: List[T forSome { type T <: Account[T] }]): List[T forSome { type T <: Account[T] }] = {
    accounts map { _.addFunds(-amount) }
  }
}
```

The important thing to notice here is that the type of individual members of
the list are existentially bounded, rather than the list being existentially
bounded as a whole. This is important, because it means that the type of
elements may vary, rather than something like `List[T] forSome { type T <: Account[T] }` 
which states that the values of the list are of some consistent
subtype of T.

So, this is a bit of an issue, but not a terrible one. The existential types
clutter up our codebase and sometimes give the type inferencer headaches, but
it's not intractable. The ability to state these existential type bounds does,
however, showcase one advantage that Scala's existentials have over Java's
wildcard types, which cannot express this same construct accurately.

The most subtle point about F-bounded types that is important to grasp is that
the type bound is *not* as tight as one would ideally want it to be; instead of
stating that a subtype must be eventually parameterized by itself, it simply
states that a subtype must be parameterized by *some (potentially other)
subtype*. Here's an example.

```scala
class MalignantAccount extends Account[SavingsAccount] {
  def addFunds(amount: BigDecimal) = new SavingsAccount(-amount)
}
```

This will compile without error, and presents a bit of a pitfall. Fortunately,
the type bounds that we were required to declare at the use sites will prevent
many of the failure scenarios that we might be concerned about:

```scala
object Test {
  def main(argv: Array[String]): Unit = {
    Account.deposit(BigDecimal("10.00"), new MalignantAccount)
  }
}

nuttycom@crash: ~/tmp/scala/f-bounds $ scalac Test.scala
Test.scala:27: error: inferred type arguments [MalignantAccount] do not conform to method deposit's type parameter bounds [T <: Account[T]]
    deposit(BigDecimal("10.00"), new MalignantAccount)
    ^
one error found
```

but it's a little disconcerting to realize that this level of strictness is
*only* available at the use site, and cannot be readily enforced (except
perhaps by implicit type witness trickery, which I've not yet tried) in the
declaration of the supertype, which is what we were really trying to do in the
first place.

Finally, it's important to note that F-bounded type polymorphism in Scala falls
on its face when you start talking about higher-kinded types. Suppose that one
desired to state a supertype for Scala's structurally monadic types (those
which can be used in for-comprehensions):

```scala
trait Monadic[M[+_] <: ({ type λ[+α] = Monadic[M, α] })#λ, +A] {
  def map[B](f: A =&gt; B): M[B]
  def flatMap[B](f: A =&gt; M[B]): M[B]
}
```

This fails to compile outright, complaining about cyclic references in the type
constructor M.

In conclusion, my experience has been that F-bounded type polymorphism is
tricky to get right and causes typing clutter in the codebase. That isn't to
say that it's without value, but I think it's best to consider very carefully
whether it is actually necessary to your particular application before you step
into the rabbit hole. Most of the time, there's no wonderland at the bottom.

<b>EDIT:</b>

Heiko asks below why not use the following definition of debitAll:

```scala
object Account {
  def debitAll2[T <: Account[T]](amount: BigDecimal, accounts: List[T]): List[T] = {
    accounts map { _.addFunds(-amount) }
  }
}
```

The problem here is that this has a subtly different meaning; this says that
for debitAll2, all the members of the list must be of the *same* subtype of
Account. This becomes apparent when we actually try to use the method with a
list where the subtype varies. In both constructions, actually, you end up
having to explicitly ascribe the type of the list, but I've not been able to
find a variant for the debitAll2 where the use site will actually compile with
such a variant-membered list.

```scala
object Test {
  def main(argv: Array[String]): Unit = {
    // compiles, though requires type ascription of the list; this is where the inferencer breaks down.
    Account.debitAll(BigDecimal("10.00"), List[T forSome { type T <: Account[T] }](new SavingsAccount(BigDecimal("0")), new BrokerageAccount(BigDecimal("0"))))

    // doesn't compile
    // Account.debitAll2(BigDecimal("10.00"), new SavingsAccount(BigDecimal("0")) :: new BrokerageAccount(BigDecimal("0")) :: Nil)

    // doesn't compile
    // Account.debitAll2(BigDecimal("10.00"), List[T forSome { type T <: Account[T] }](new SavingsAccount(BigDecimal("0")), new BrokerageAccount(BigDecimal("0"))))
  }
}
```

*migrated from http://logji.blogspot.com/2012/11/f-bounded-type-polymorphism-give-up-now.html *

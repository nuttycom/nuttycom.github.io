---
title: The Abstract Future
---

*This post is being resurrected from the dustbin of history - it was originally
posted on the precog.com engineering blog, which has since been lost to
acquisition and bitrot. My opinion on the applicability of the following
techniques has changed somewhat since I originally wrote this post; I will
address this in a followup post in the near future. Briefly, though, I believe
that parameterizing each method with an implicit monad constraint is preferable
where possible; it provides the user with greater flexibility.*

In our last blog post on Precog development, Daniel wrote about how we use the
Cake Pattern to structure our codebase and to leave the implementation types
abstract as long as possible. As he showed in that post, this is an extremely
powerful concept; by keeping a type existential, values of that type remain
opaque to any modules that aren’t “aware” of the eventual type chosen, and so
are prevented by the compiler from breaking encapsulation boundaries.

In today’s post, we’re going to extend this notion beyond types to handle type
constructors, and in so doing will show a mechanism that allows us to switch
out entire models of computation.

If you’ve been working with Scala for any length of time, you’ve undoubtedly
heard the word “monad” floating around in one context or another, perhaps in a
discussion about the syntactic sugar provided by Scala’s ‘for’ keyword or a
blog post discussing how the Option type can be used to avoid the pitfalls of
null references. While a significant amount of discussion of monads in Scala
focuses on the “container” types, a few types common in the Scala ecosystem
display a more interesting facet of monadic composition – delimited
computation. While all monadic types exhibit this in composition, perhaps the
most commonly used monadic type in Scala that exemplifies this sort of use
directly is `akka.dispatch.Future`, (which is scheduled to replace Scala’s
current `Future` interface in the standard library in Scala 2.10) which encodes
asynchronous computation.  It embodies the aspect of monadic composition that
we’re most concerned with here by providing a flexible way to order the steps
of a computation.

I’d like to step back a moment here and state that this post isn’t intended to
function as a monad tutorial; there are numerous (perhaps too many!) articles
about monads, and their relevance to programming in Scala exist elsewhere. If
you’re new to the concept it will be useful for you to take advantage of one or
more of these resources before continuing here. It is, however, important to
point out at first that the use of monads in Scala, while pervasive (as
evidenced by the presence of ‘for’ as syntactic sugar for monadic composition)
is somewhat idiosyncratic in that the Scala standard libraries actually provide
no Monad type. For this, we have to look outside of the standard library to the
excellent scalaz project. Scalaz’s encoding of the monadic abstraction relies
upon the implicit typeclass pattern. The base Monad type is shown here,
simplified, for reference:

```scala
trait Monad[M[_]] {
  def point[A](a: => A): M[A]
  def bind[A, B](m: M[A])(f: A => M[B]): M[B]
  def map[A, B](m: M[A])(f: A => B): M[B] = bind(m)(a => point(f(a))) 
}
```

You’ll note that the Monad trait is not parameterized by a specific type, but
instead a type constructor of one argument. The methods defined inside of Monad
are then parametrically polymorphic, which means that they must provide a
specific type to be inserted into the “hole” at the invocation point. This will
be important later, when we talk about how to actually take advantage of this
abstraction.

Scalaz provides implementations of this type for most of the monadic types in
the Scala standard library, as well as several more sophisticated monadic
types, which we’ll return to in a moment. For now, however let’s talk a bit
about Akka’s Futures.

An Akka Future represents a computation whose value is produced asynchronously,
and which may fail. Also, as I noted before, akka.dispatch.Future is monadic;
that is, it is a type for which the Monad trait above can be trivially
implemented and which satisfies the monad laws, and so it provides an extremely
useful primitive for composing asynchronous computations without all sorts of
tedious mucking about with manual management of threads and shared mutable
state. At Precog, we use Futures extensively, both in a direct fashion and to
allow us a composable way to interact with subsystems that are implemented atop
Akka’s actor framework. Futures are arguably one of the best tools we have for
reining in the complexity of asynchronous programming, and so our many of our
early versions of APIs in our codebase exposed Futures directly. For example,
here’s a snippet of one of our internal APIs, which follows the Cake pattern as
described previously.

```scala
trait DatasetModule {
  type Dataset 

  trait DatasetLike {
    /** The members of this dataset will be used to determine what sets to
        load, and the resulting sets will be unioned together */
    def load: Future[Dataset]

    /** Sorts the dataset by the specified value function. */
    def sort(sortBy: /*...*/): Future[Dataset]

    /** Retains a prefix of this dataset. */
    def take(size: Int): Dataset

    /** Map members of the dataset into the A type using the specified value 
        function, then combine using the resulting monoid */
    def reduce[A: Monoid](mapTo: /*...*/): Future[A]
  }
}
```

The Dataset type here is something of a strawman, but is loosely representative
of the type that we use internally to represent an intermediate result of a
computation - a lazy data structure with a number of operations that can be
used to manipulate it, some of which may involve actually evaluating a function
over the entire dataset and which may involve I/O, distributed evaluation, and
asynchronous computation. Based on this interface, it’s easy to see that
evaluation of some query with respect to a dataset might involve a load, a
sort, taking a prefix, and a reduction of that prefix. Moreover, such an
evaluation will not rely upon anything except the monadic nature of Future to
compose its steps. What this means is that from the perspective of the consumer
of the DatasetModule interface, the only aspect of Future that we’re relying
upon is the ability to order operations in a statically checked fashion; the
sequencing, rather than any particular semantics related to Future’s
asynchrony, is the relevant piece of information provided by the type. So, the
following generalization becomes natural:

```scala
trait DatasetModule[M[+_]] {
  type Dataset 
  implicit def M: Monad[M]

  trait DatasetLike {
    /** The members of this dataset will be used to determine what sets to
        load, and the resulting sets will be unioned together */
    def load: M[Dataset]

    /** Sorts the dataset by the specified value function. */
    def sort(sortBy: /*...*/): M[Dataset]

    /** Retains a prefix of this dataset. */
    def take(size: Int): Dataset

    /** Map members of the dataset into the A type using the specified value 
        function, then combine using the resulting monoid */
    def reduce[A: Monoid](mapTo: /*...*/): M[A]
  }
}
```

and, of course, down the road some concrete implementation of DatasetModule
will refine the type constructor M to be Future:

```scala
/** The implicit ExecutionContext is necessary for the implementation of M.point */
class FutureMonad(implicit executor: ExecutionContext) extends Monad[Future] {
  override def point[A](a: => A): Future[A] = Future { a }
  override def bind[A, B](m: Future[A])(f: A => Future[B]): Future[B] = 
    m flatMap f
}

abstract class ConcreteDatasetModule(implicit executor: ExecutionContext) 
extends DatasetModule[Future] {
  val M: Monad[Future] = new FutureMonad 
}
```

In practice, we may actually leave `M` abstract until “the end of the
universe.” In the Precog codebase, the `M` type will frequently represent the
bottom of a stack of monad transformers including `StateT`, `StreamT`,
`EitherT` and others that the actual implementation of the `Dataset` type
depends upon.

This generalization has numerous benefits. First, as with the previous examples
of our use of the Cake pattern, consumers of the DatasetModule trait are
completely and statically insulated from irrelevant details of the
implementation type. An important such consumer is a test suite. In a test, we
probably don’t want to worry about the fact that the computation is being
performed asynchronously; all that we care about is that we obtain a correct
result. If our M is in fact at the bottom of a transformer stack, we can
trivially replace it with the identity monad and use the “copointed” nature of
this monad (the ability to “extract” a value from the monadic context). This
allows us to build a similarly generic test harness:

```scala
/** Copointed is available from scalaz as well; reproduced here for clarity */
trait Copointed[M[_]] {
  /** Extract and return the value from the enclosing context. */
  def copoint[A](m: M[A]): A
}

trait TestDatasetModule[M[+_]] extends DatasetModule {
  implicit def M: Monad[M] with Copointed[M]

  //... utilities for test dataset generation, stubbing load/sort, etc.

}
```

For most cases, we’ll use the identity monad for testing. Suppose that we’re
testing the piece of functionality described earlier, which has computed a
result from the combination of a load, a sort, take and reduce. The test
framework need never consider the monad that it’s operating in.

```scala
import scalaz._
import scalaz.syntax.monad._
import scalaz.syntax.copointed._

class MyEvaluationSpec extends Specification {
  val module = new TestDatasetModule[Id] with ConcreteDatasetModule[Id] { 
    val M = Monad[Id] // the monad for Id is copointed in Scalaz.
  }
  
  “evaluation” should {
    “determine the correct result for the load/sort/take/reduce case” in {
      val loadFrom: module.Dataset = //...
      val expected: Int = //...

      val result = for {
        ds 
        sorted - ds.sortBy(mySortFun)
        prefix = sorted.take(10)
        value - prefix.reduce[Int]myCountFunc)
      } yield value

      result.copoint must_== expected
    }
  }
}
```

In the case that we have a portion of the implementation that actually depends
upon the specific monadic type (say, for example, that our sort implementation
relies on Akka actors and the “ask” pattern under the hood, so that we’re using
Futures) we can simply encode this in our test in a straightforward fashion:

```scala
abstract class TestFutureDatasetModule(implicit executor: ExecutionContext)
extends TestDatasetModule[Future] {
  def testTimeout: akka.util.Duration

  object M extends FutureMonad(executor) with Copointed[Future] {
    def copoint[A](m: Future[A]): A = Await.result(m, testTimeout)
  }
}
```

Future is, of course, not properly copointed (since Await can throw an
exception) but for the purposes of testing (and testing exclusively) this
construction is ideal. As before, we get exactly the type that we need,
statically determined, at exactly the place that we need it.

In practice, we’ve found that abstracting away the particular monad that our
code is concerned with has aided tremendously with keeping the concerns of
different parts of our codebase well isolated, and ensuring that we’re simply
not able to sidestep the sequencing requirements that are necessary to make a
large, functional codebase work together as a coherent whole. As an added
benefit, many parts of our application that were not initially designed
thinking in terms of parallel execution are able to execute concurrently. For
example, in many cases we’ll be computing a `List[M[...]]` and then using the
`sequence` function provided by `scalaz.Traverse` to turn this into an
`M[List[...]]` - and when `M` is future, each element may be computed in
parallel, with the final sequenced result becoming available only when all the
computations to produce the members of the list are complete. And, ultimately,

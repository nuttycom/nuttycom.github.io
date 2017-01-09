---
title: On Extraction
---

Scala case classes are great; they give you a tremendous leg up when it comes
to implementing pattern matching. However, there's a drawback: the unapply
method of case class companion objects take arguments of the type of the case
class, and return `Option[TupleN[...]]` where `N` is the number of variables to the
case class constructor. What's more, it's not possible to overload or override
the unapply method to deconstruct other types of by creating your own
companion object on the side, as shown in this REPL session:

```scala
scala> case class  Foo(val i: Int, val s: String)
defined class Foo

scala> object Foo {
     |   def unapply(s: String): Option[Foo] = Some(new Foo(s.substring(0, 2).toInt, s.substring(2)))
     | }
defined module Foo

scala> "12abcd" match {
     |   case Foo(i, s) => println(s, i)
     | }
<console>:10: error: wrong number of arguments for object Foo
         case Foo(i, s) => println(s, i)
              ^
<console>:10: error: wrong number of arguments for method println: (Any)Unit
         case Foo(i, s) => println(s, i)
```

Sadly, it doesn't even seem to be possible to move this extractor to another
class and get the desired result:

```scala
scala> object SFoo {
     |   def unapply(s: String): Option[Foo] = Some(new Foo(s.substring(0, 2).toInt, s.substring(2)))
     | }
defined module SFoo

scala> "12abcd" match {
     |   case SFoo(i, s) => println(s, i)
     | }
<console>:9: error: wrong number of arguments for object SFoo
         case SFoo(i, s) => println(s, i)
              ^
<console>:9: error: wrong number of arguments for method println: (Any)Unit
         case SFoo(i, s) => println(s, i)

```

Fortunately, there is a way around this issue: simply define your own class
that extends ProductN! Then, you can define as many extractors as you want in
different pseudo-companion objects and use them idiomatically:

```scala
scala> class Foo(val i: Int, val s: String) extends Product2[Int, String] {
     |   val (_1, _2) = (i, s)
     | }
defined class Foo

scala> object Foo {
     |   def unapply(f: Foo): Option[(Int, String)] = Some((f.i, f.s))
     | }
defined module Foo

scala> object SFoo {
     |   def unapply(s: String): Option[Foo] = Some(new Foo(s.substring(0, 2).toInt, s.substring(2)))
     | }
defined module SFoo

scala> "12abcd" match {
     |   case SFoo(i, s) => println(s, i)
     | }
(abcd,12)

scala> new Foo(1, "hi") match {
     |   case Foo(i, s) => println(s, i)
     | }
(hi,1)

```

Also, naturally, the trivial extractor (Foo => Foo) can supplant the Foo =>
Tuple2 extractor above:

```scala
scala> object Foo {
     |   def unapply(f: Foo): Option[Foo] = Some(f)
     | }

scala> new Foo(1, "hi") match {
     |   case Foo(i, s) => println(s, i)
     | }
(hi,1)
```

*migrated from http://logji.blogspot.com/2010/02/deconstructables.html *

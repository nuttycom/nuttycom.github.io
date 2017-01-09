---
title: The Catamorphism Challenge
---

My solution to Tony's [catamorphism challenge](http://blog.tmorris.net/debut-with-a-catamorphism/):

```scala
trait MyOption[+A] {
  import MyOption._

  def cata[X](some: A => X, none: => X): X
  def map[B](f: A => B): MyOption[B] = cata(a => some(f(a)), none[B])
  def flatMap[B](f: A => MyOption[B]): MyOption[B] = cata(f, none[B])
  def getOrElse[AA >: A](e: => AA): AA = cata(a => a, e)
  def filter(p: A => Boolean): MyOption[A] = cata(a => if p(a) some(a) else none[A], this)
  def foreach(f: A => Unit): Unit = cata(f, ())
  def isDefined: Boolean = cata(_ => true, false)
  def isEmpty: Boolean = cata(_ => false, true)
  def get: A = cata(a => a, error("get on none"))
  def orElse[AA >: A](o: MyOption[AA]): MyOption[AA] = cata(a => some(a), o)
  def toLeft[X](right: => X): Either[A, X] = cata(a => Left(a), Right(right))
  def toRight[X](left: => X): Either[X, A] = cata(a => Right(a), Left(left))
  def toList: List[A] = cata(a => List(a), Nil)
  def iterator: Iterator[A] = toList.elements
}

object MyOption {
  def none[A] = new MyOption[A] {
    def cata[X](s: A => X, n: => X) = n
  }

  def some[A](a: A) = new MyOption[A] {
    def cata[X](s: A => X, n: => X) = s(a)
  }
}
```

*migrated from http://logji.blogspot.com/2009/12/catamorphisms-easy-peasy.html *

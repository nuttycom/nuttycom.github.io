---
title: Keyword Escape in Scala
---

James Iry just mentioned something I'd never noticed before in #scala:

```scala
scala> def f(i: Int, x: Int) = i match { case `x` => println("x!"); case _ => println("nope") }
f: (i: Int,x: Int)Unit

scala> f(1, 1)
x!

scala> f(1, 2)
nope
```

That's one of those things that I'd idly wondered how to do, but hadn't had a
use case so hadn't really gone looking. Good to know it's possible and
simple... and more or less makes sense.  More or less. Actually, I guess I'm
kind of glad that the following doesn't compile:

```scala
scala> def f(i: Int) = i match { case `yield` => println(`yield`) }
<console>:4: error: not found: value yield
       def f(i: Int) = i match { case `yield` => println(`yield`) }
```

I can live with the restriction as not being able to use a reserved word for a
variable on the lhs of a pattern match, though it does make for a weird
inconsistency:

```scala
scala> val (`yield`, x) = (1, 1)
<console>:4: error: not found: value yield
       val (`yield`, x) = (1, 1)
            ^

scala> val `yield` = 1
yield: Int = 1
```

Chalk up one more to the box of "pain points caused by using pattern matching
for parallel assignment."

*migrated from http://logji.blogspot.com/2010/02/trick-of-day.html *


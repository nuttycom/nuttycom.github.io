---
title: Correcting the Visitor Pattern
---

I'm writing this post because I want to address a problem that I see time and
time again: people who are trying to figure out how to encode algebraic data
types in languages that do not support them come up with [all kinds of crazy
solutions](http://shahriarhaque.com/blog/blog/structural-pattern-matching-using-exceptions/).
But, there is a simple and effective encoding of algebraic data types that
everyone knows about, but had just been doing wrong: the Visitor pattern.  At
this point, I expect many devotees of functional programming to start screaming
"but... mutability!!!" And, yes, the mutability required by the textbook
definition of the Visitor pattern is indeed a major problem - the problem that
I intend to show how to correct right here. The solution is remarkably simple.
Here's a tree traversal using the Visitor pattern, implemented *correctly* in
Java:

```java
public interface Tree<A> {
  public <B> B accept(TreeVisitor<A, B> v);
}

public class Empty<A> implements Tree<A> {
  public <B> B accept(TreeVisitor<A, B> v) {
    return v.visitEmpty();
  }
}

public class Leaf<A> implements Tree<A> {
  public final A value;

  public Leaf(A value) {
    this.value = value;
  }

  public <B> B accept(TreeVisitor<A, B> v) {
    return v.visitLeaf(this);
  }
}

public class Node<A> implements Tree<A> {
  public final Tree<A> left;
  public final Tree<A> right;

  public Node(Tree<A> left, Tree<A> right) {
    this.left = left;
    this.right = right;
  }

  public <B> B accept(TreeVisitor<A, B> v) {
    return v.visitNode(this);
  }
}

public interface TreeVisitor<A, B> {
  public B visitEmpty();
  public B visitLeaf(Leaf<A> t);
  public B visitNode(Node<A> t);
}
```

This is exactly the traditional Visitor pattern, with one minor variation: the
`accept` method is generic (parametrically polymorphic), returning whatever
return type is defined by a particular `TreeVisitor` instance instead of void.
This is a vitally important distinction; by making accept polymorphic and
non-void returning, it allows you to escape the curse of being forced to rely
on mutability to accumulate a result. Here's an example of the implementation
of the `depth` method from the previously mentioned blog post, and an example
of its use. You'll note that no mutable variables were harmed (or indeed used)
in the creation of this example:

```java
public class TreeUtil {
  public static final <A> TreeVisitor<A, Integer> depth() {
    return new TreeVisitor<A, Integer>() {
      public Integer visitEmpty() {
        return 0;
      }

      public Integer visitLeaf(Leaf<A> t) {
        return 1;
      }

      public Integer visitNode(Node<A> t) {
        int leftDepth = t.left.accept(this);
        int rightDepth = t.right.accept(this);
        return (leftDepth > rightDepth) ? leftDepth + 1 : rightDepth + 1;
      }
    };
  }
}

public class Example {
  public static void main(String[] argv) {
    Tree<String> leftBiased = new Node<String>(
      new Node<String>(
        new Node<String>(
          new Leaf<String>("hello"),
          new Empty<String>()
        ),
        new Leaf<String>("world")
      ),
      new Empty<String>()
    );

    assert(leftBiased.accept(TreeUtil.<String>depth()) == 3);
  }
}
```

TreeVisitor encodes the *f-algebra* for the Tree data type; the accept method
is the *catamorphism* for Tree. Moreover, given this definition, you can also
see that [the visitor forms a monad](./2009-12-08-monadic_visitor.html) (example
in Scala), giving rise to lots of nice compositional properties.  Implemented
in this fashion, Visitor is actually nothing more (and nothing less) than a
multiple dispatch function over the algebraic data type in question. So stop
returning void from your visitors, and ramp up their power in the process!

*migrated from http://logji.blogspot.com/2012/02/correcting-visitor-pattern.html *

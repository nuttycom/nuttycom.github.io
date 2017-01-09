---
title: Composable Bindings in Lift Part II
---

In my [previous post](./2009-09-29-composable_bindings_in_lift.html) I presented
the DataBinding[T] type (T => NodeSeq => NodeSeq) and illustrated how classes
inhabiting this type could be used along with an implicit binder function to
create modular, reusable rendering logic for objects. In this post, I will
discuss how to assemble these components and strategies for handling rendering
over inheritance hierarchies.

I left off last time with this DataBinding trait:

```scala
trait OrderBinding extends DataBinding[Order] {
  implicit val userBinding: DataBinding[User]
  implicit val addressBinding: DataBinding[Address]
  implicit val lineBinding: DataBinding[OrderLine]
  implicit val txnBinding: DataBinding[Transaction]

  def apply(order: Order): Binding = (xhtml: NodeSeq) => {
    val itemTemplate = chooseTemplate("order", "lineItem", xhtml)
    val txnTemplate = chooseTemplate("order", "transaction", xhtml)

    bind("order", xhtml,
      "user" -> order.user.bind("templates-hidden/user/order-display".path),
      "shipTo" -> order.shipTo.bind("templates-hidden/address/order-display".path),
      "lineItems" -> order.items.flatMap(_.bind(itemTemplate)).toSeq,
      "transactions" -> order.transactions.flatMap(_.bind(txnTemplate)).toSeq
    )
  }
}
```

This `DataBinding[Order]` is a trait, not an object like the previous binding
instances. In addition, the other DataBinding instances that it delegates to
are abstract implicit vals - meaning that I will need to create a concrete
implementation of this trait to actually use it. The interesting thing here is
that these vals are declared implicit within the scope of the trait, but do not
need to be declared implicit when the trait is extended and made concrete. I
can then compose what I want my binding to look like at the use site (say an
ordinary snippet) like this:

```scala
import Bindings._

class MySnippet {
  private implicit object ShowOrderBinding extends OrderBinding {
    val userBinding = UserBinding
    val addressBinding = MailingAddressBinding
    val lineBinding = OrderLineBinding
    val txnBinding = TransactionBinding //we'll implement this in a minute
  }

  def showOrder(xhtml: NodeSeq): NodeSeq = {
    val order: Order = //get an order from somewhere
    order.bind(xhtml)
  }
}
```

The `ShowOrderBinding` selects the bindings I want for the components of the
order in a declarative fashion, and since it is an implicit object in scope, it
is selected as the DataBinding instance to render the order when order.bind is
called. If I decide that I want to use the AdminAddressBinding from the
previous post instead, I simply substitute it for MailingAddressBinding at the
use site and I'm done!

There's one more thing that needs to be done in the above code - to implement
`TransactionBinding`. This is a little tricky because there are several
subclasses of `Transaction` that may be in the list of transactions for an
order, so I'm going to need to recover type information about those instances
at runtime in order to be able to choose the correct binding for a given
`Transaction`. Furthermore, there's some information in the base `Transaction`
type that I don't want to have to repeat bindings for in each subtype binding,
so I'm going to use function composition to "add together" two different
bindings.

There are two different approaches I'm going to show for how one might go about
doing this: first, I'll try using a straightforward Scala match statement to
figure out the object's type. That looks like this:

```scala
trait TransactionBinding extends DataBinding[Transaction] {
  implicit val authBinding: DataBinding[AuthTransaction]
  implicit val captureBinding: DataBinding[CaptureTransaction]
  implicit val shipBinig: DataBinding[ShipTransaction]

  def apply(txn: Transaction): Binding = {
    val base: Binding = bind("txn", _,
      "scheduleDate" -> Text(txn.scheduledAt.toString),
      "completionDate" -> Text(txn.completedAt.getOrElse("(incomplete)").toString),
      "state" -> Text(
        txn.state match {
          case COMPLETE => "complete"
          case _ => "weird"
        }
      )
    )

    base compose (
      txn match {
        case auth: AuthTransaction => auth.binding
        case capture: CaptureTransaction => capture.binding
        case ship: ShipTransaction => ship.binding
        case _ => error("Unrecognized transaction type!")
      }
    )
  }
}
```

Here, I've created a base binding that defines some simple behavior for how to
bind elements that are common to all transactions. I then *compose* that
generated binding with a binding for the appropriate subclass. The
determination of the subclass binding is of necessity a bit boilerplate-laden,
but that's what happens when one throws away compile-time type information. The
alternative, of course, would be to store each type of transaction in a
separate collection associated with the order, but that's not the way I've gone
for now.

There is, however, a hidden bug that may lurk in the pattern match above,
depending upon your environment. Let's supposed for a minute that our model
classes are persisted by Hibernate - and that the list of Transactions is
annotated such that it is lazily, instead of eagerly, fetched from the
database. In this case, the transactions returned may be translucently wrapped
in runtime proxies. I say translucently instead of transparently because there
is one critical respect in which a proxy does not behave in the same manner as
the class being proxied - how it responds to the instanceof check that
underlies the Scala pattern match. In this case, we will need to use a
`Visitor` to correctly get the type for which we want to delegate binding. To
do this, we'll modify our transaction classes as so:

```scala
trait TransactionFunc[T] {
  def apply(auth: AuthTransaction): T
  def apply(capture: CaptureTransaction): T
  def apply(ship: ShipTransaction): T
}

abstract class Transaction(val scheduledAt: DateTime, val completedAt: Option[DateTime], state: State) {
  def accept[T](f: TransactionFunc[T]): T
}

case class AuthTransaction(cc: CreditCard, amount: BigDecimal, scheduledAt: DateTime, completedAt: Option[DateTime], state: State) extends Transaction(scheduledAt, completedAt) {
  def accept[T](f: TransactionFunc[T]): T = f.apply(this)
}

case class CaptureTransaction(auth: AuthTransaction, ...) extends Transaction(...)  {
  def accept[T](f: TransactionFunc[T]): T = f.apply(this)
}

case class ShipTransaction(items: List[OrderLine], ...) extends Transaction(...)  {
  def accept[T](f: TransactionFunc[T]): T = f.apply(this)
}
```


Then, the implementation of our transaction binding becomes this:

```scala
trait TransactionBinding extends DataBinding[Transaction] {
  implicit val authBinding: DataBinding[AuthTransaction]
  implicit val captureBinding: DataBinding[CaptureTransaction]
  implicit val shipBinig: DataBinding[ShipTransaction]

  def apply(txn: Transaction): Binding = {
    val base: Binding = //same as before...

    base compose txn.accept(
      new TransactionFunc[Binding] {
        def apply(auth: AuthTransaction) = auth.binding
        def apply(capture: CaptureTransaction) = capture.binding
        def apply(ship: ShipTransaction) = ship.binding
      }
    )
  }
}
```

Since the correct overloading of TransactionFunc.apply will be statically
determined, there is no possibility of a proxy getting in the way of our
binding. 

So, now we have a set of tools for binding composition. Here are some
additional ideas of where this might be able to go, given the appropriate
changes to Lift.

In my example of the `showOrder` snippet above, the implementation is
trivial... so trivial, in fact, that it almost doesn't need to be there. If the
notion of a binding was fully integrated into Lift, we could make Loc somewhat
more powerful. Loc is one of the primary classes in Lift used to generate the
SiteMap, which declares what paths are accessible within a Lift application. A
path in a Loc typically corresponds to the path to an XML template that will
specify the snippets and markup for a request. An important thing to note
though is that `Loc` is not simply `Loc` - it is in fact `Loc[T]`, where by
default `T` is `NullLocParams`, a meaningless object essentially equivalent to
`Unit`.

When one looks at making `T` something other than `Unit`, `Loc` starts to become
significantly more interesting - first, it declares a rewrite method that can
be overriden to populate the `Loc` with an instance of `T` as part of its
processing, and secondly, `Loc` can specify explicitly the template to be used to
render the `Loc`, instead of using the one that corresponds to the path. Wait...
we have an instance, and we have a template... what if we could specify a
`DataBinding[T]` as well? In this case, we might not need for there to be a
snippet at all!


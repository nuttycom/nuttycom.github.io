---
title: Composable Bindings in Lift
---

I've been using [Lift](http://liftweb.net) as the framework behind a small
internal web application at my company for the past year or so. The experience
has been generally positive, despite a few quibbles I have with how state is
typically maintained and passed around within an application. 

Lift takes a "view first" approach to building web applications, as opposed to
the commonly favored MVC pattern. In practice, "view first" boils down to
essentially this: when you create an XHTML template for a page, that template
will contain one or more XML tags where the name of the tag refers to a method
to be called with the body of the tag as an argument. Such a method will return
the XHTML to be substituted in place of the original tag and its contents. In
Lift parlance such a method is called a "snippet," and has the type NodeSeq =>
NodeSeq. In order to more fully understand the rest of this article, I
recommend you read more about snippets
[here](http://liftweb.net/docs/getting_started/mod_master.html#x1-110002.5) if
you're not already familiar with them.

The interesting thing about this signature is that functions with this type can
easily be composed. What's more, the method most frequently used to implement
snippets, BindHelpers.bind, can be partially applied to its arguments to give a
closure with the same signature. 

When I'm designing a website, most of the time rendering a page (or even a
snippet within a page) isn't about displaying the data from a single object -
instead, I often want to render a whole object graph. Ideally, I want to be
able to choose a rendering for each type of object I'm concerned with, then
compose these elements to produce the final page. Templates for objects should
be reusable and modular, and for any given page I should be able to pick
between multiple renderings of any given object, in some places displaying a
concise summary and elsewhere a detailed exposition of all the available data. 

Furthermore, while I believe that "the meaning should belong with the bytes" I
also think that the rendering should be as decoupled from the bytes as possible
to allow for maximum flexibility. Hence, I don't want my model classes to have
any sort of "render" method - the display is something that should be layered
on above the semantics provided by the names and types of the members of the
object.

With these goals and composition in mind, I started with the following tiny bit
of code:

```scala

import net.liftweb.http.TemplateFinder.findAnyTemplate
import net.liftweb.util.{Box,Full,Empty,Failure,Log}
import scala.xml._

object Bindings {
  type Binding = NodeSeq => NodeSeq

  type DataBinding[T] = T => NodeSeq => NodeSeq

  //adds a bind() function to an object if an implicit DataBinding is available for that object
  implicit def binder[T](t: T)(implicit binding: DataBinding[T]): Binder = Binder(binding(t))
  implicit def binder(binding: Binding): Binder = Binder(binding)

  //decorator for a binding function that allows it to be called as bind() rather than apply()
  //and also provides facilities for binding to a specific template
  case class Binder(val binding: Binding) {
  def bind(xhtml: NodeSeq): NodeSeq = binding.apply(xhtml)
    def bind(templatePath: List[String]): NodeSeq = {
      findAnyTemplate(templatePath) map binding match {
        case Full(xhtml) => xhtml
        case Failure(msg, ex, _) => Text(ex.map(_.getMessage).openOr(msg))
        case Empty => Text("Unable to find template with path " + templatePath.mkString("/", "/", ""))
      }
    }
  }

  object NullBinding extends Binding {
    override def apply(xhtml : NodeSeq) : NodeSeq = NodeSeq.Empty
  }

  object StringBinding extends DataBinding[String] {
    override def apply(msg: String) = (xhtml: NodeSeq) => Text(msg)
  }

  implicit def path2list(s: String) = new {
    def path: List[String] = s.split("/").toList
  }
}
```

The principle idea is that in my application, I will create one or more
DataBinding instances for each of my model types, and that then by simply
declaring an implicit val containing the instance I want in the narrowest scope
possible, I can call .bind on any object for which an implicit DataBinding is
available, or .binding if I want to compose multiple such binding functions and
close over the model instance. Furthermore, this gives me good ways to compose
both over composition and inheritance hierarchies, as I'll show below. 

Consider that I have the following simplified, store-oriented model objects. In
Lift, these would probably be implemented using Mapper or JPA - in the app that
I've been working on, these are Java classes persisted by Hibernate - certainly
not things that I want to change in order to support rendering by Scala.

```scala
case class User(fname: String, lname: String, email: String, joinedOn: DateTime)
case class Address(
  recipient: String, 
  addr1: String, 
  addr2: String, 
  city: String, 
  state: String, 
  country: String, 
  postalCode: String
)

case class Product(name: String, basePrice: BigDecimal, shippingCost: Option[BigDecimal])

case class OrderLine(product: Product, quantity: Int)

class Order(
  val user: User, 
  val placedOn: DateTime, 
  val shipTo: Address, 
  var items: List[OrderLine], 
  var transactions: List[Transaction]
)

sealed trait State
object PENDING extends State
object IN_PROGRESS extends State
object COMPLETE extends State
object FAILED extends State
object CANCELED extends State

abstract class Transaction(val scheduledAt: DateTime, val completedAt: Option[DateTime], state: State)
case class AuthTransaction(
  cc: CreditCard, 
  amount: BigDecimal, 
  scheduledAt: DateTime, 
  completedAt: Option[DateTime], 
  state: State
) extends Transaction(scheduledAt, completedAt)
case class CaptureTransaction(auth: AuthTransaction, ...) extends Transaction(...)
case class ShipTransaction(items: List[OrderLine], ...) extends Transaction(...)
```

With this set of models, I have a few different ways that they will be 
rendered - the display to the users will not be the same as the display 
to site administrators, I may (or may not) want to reuse the same rendering
code for a line item in an order as in a shipping transaction, and so forth.
Let's take a look at how this ends up.

First, I want to create some DataBinding objects for my classes that are built
just on primitive types. Let's start with User.

```scala

import net.liftweb.util.Helpers._ //this is where BindHelpers.bind comes from
import scala.xml._
import Bindings._

object UserBinding extends DataBinding[User] {
  def apply(user: User): Binding = bind("user", _,
    "name" -> Text(user.fname+" "+user.lname),
    "email" -> Text(user.email)
  )
}

object MailingAddressBinding extends DataBinding[Address] {
  def apply(addr: Address): Binding = bind("address", _,
    "line1" -> Text(addr.recipient),
    "line2" -> Text(addr.addr1),
    "line3" -> Text(addr.addr2),
    "line4" -> Text(addr.city+", "+addr.state+" "+addr.postalCode+" "+addr.country)
  )
}

object AdminAddressBinding extends DataBinding[Address] {
  def apply(addr: Address): Binding = bind("address", _,
    "recipient" -> Text(addr.recipient),
    // imagine more literal bindings like the previous one here
  )
}

object OrderLineBinding extends DataBinding[OrderLine] {
  //more of the same
}
```

These implementations take advantage of partial application and the Scala type
inferencer to turn the call to bind(...) into a closure with the type Binding,
which is `NodeSeq => NodeSeq`. At this point, we're not taking advantage of the
fact that these functions can compose, so we'll do so now a we build bindings
for our more complex objects. 

```scala

//same imports as before

trait OrderBinding extends DataBinding[Order] {
  implicit val userBinding: DataBinding[User]
  implicit val addressBinding: DataBinding[Address]
  implicit val lineBinding: DataBinding[OrderLine]
  implicit val txnBinding: DataBinding[Transaction]

  def apply(order: Order): Binding = (xhtml: NodeSeq) => {
    val itemTemplate = chooseTemplate("order", "lineItem", xhtml)
    val txnTemplate = chooseTemplate("order", "transaction", xhtml)

    bind(
      "order", xhtml,
      "user" -> order.user.bind("templates-hidden/user/order-display".path),
      "shipTo" -> order.shipTo.bind("templates-hidden/address/order-display".path),
      "lineItems" -> order.items.flatMap(_.bind(itemTemplate)).toSeq,
      "transactions" -> order.transactions.flatMap(_.bind(txnTemplate)).toSeq
    )
  }
}
```

There's a lot more going on in this one. First off, we define our DataBinding
as a trait rather than a concrete object, in order to take advantage of
abstract implicit val declarations. When we go to instantiate this trait, we
will provide concrete implementations of the various bindings that we want
applied to the objects from which an Order instance is composed. The compiler
will then be able to apply the "binder" conversion to get a Binder instance
which closes over the instance with the appropriate DataBinding so that we can
simply call instance.bind and pass either a path to a template, or a chunk of
XHTML extracted from the input with chooseTemplate to get our final value.

To be continued...

*migrated from http://logji.blogspot.com/2009/09/composable-bindings-in-lift.html *

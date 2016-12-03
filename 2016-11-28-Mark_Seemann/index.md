
# Easy domain modelling with types #

**By Mark Seemann**

*Original post dated 2016-11-28 available at http://blog.ploeh.dk/2016/11/28/easy-domain-modelling-with-types/*
*All text and code copyright (c) 2016 by Mark Seemann. Used with permission.*


*Algebraic data types make domain modelling easy.*

People often ask me if I think that F# is a good general-purpose language, and when I emphatically answer yes!, the natural next question is: *why*?

For years, I've been able to answer this question in the abstract, but I've been looking for a good concrete example with which I could illustrate the answer.
I believe that I've now found such an example.

The abstract answer, by the way, is that F# has [algebraic data types](https://en.wikipedia.org/wiki/Algebraic_data_type),
which makes domain modelling much easier than in languages that don't have such types.
Don't worry if the word 'algebraic' sounds scary, though. It's not at all difficult to understand, and I'll show you a simple example.

## Payment types

At the moment, I'm working on an integration project:
I'm developing a RESTful API that serves as Facade in front of a third-party payment provider.
The third-party provider exposes their own API and web-based GUI that enable our end users to pay for services using credit cards,
PayPal, and so on. The API that I'm developing presents a simplified, RESTful API to other clients in our organisation.

The example you're going to see here is real code that I'm writing in order to implement the desired functionality.

The system must be able to handle several different types of payment:

* Sometimes, a user pays for a single thing, and that's the end of that transaction.
* Other times, however, a user engages into a long-term payment relationship. This could be, for example, a subscription, or an 'auto-fill' style of relationship. This is handled in two distinct phases:
  * An initial payment (can sometimes be for a zero amount) that authorises the merchant to make further transactions.
  * Subsequent payments, based off that initial payment. These payments can be automated, because they require no further user interaction than the initial authorisation.

The third-party service calls these 'long-term relationship' payments for *recurring* payments, but in order to distinguish between
the first and the subsequent payments in such a relationship, I decided to call them *parent and child payments*; accordingly, I call the one-off payments *individual* payments.
 
You can indicate the type of payment when interacting with the payment service's JSON-based API, like this:

```
{
  ...
  "StartRecurrent": "false"
  ...
}
```

Obviously, as the (illegal) ellipses suggests, there's much more data associated with a payment, but that's not important in this example.

Since `StartRecurrent` is `false`, this is either an individual payment, or a child payment.
If you want to start a long-term relationship, you must create a parent payment and set `StartRecurrent` to true.

Child payments, however, are a bit different, because you have to tell the payment service about the parent payment:

```
{
  ...
  "OriginalTransactionKey": "1234ABCD",
  "StartRecurrent": "false"
  ...
}
```

As you can see, when making a child payment, you supply the transaction ID for the parent payment.
(This ID is given to you by the payment service when you initiate the parent payment.)

In this case, you're clearly *not* starting a new recurrent transaction.

There are two dimensions of variation in this example: `StartRecurrent` and `OriginalTransactionKey`. Let's put them in a table:

| . | "StartRecurrent" : "false" | "StartRecurrent" : "true" |
| "OriginalTransactionKey" : null | Individual | Parent |
| "OriginalTransactionKey" : "1234ABCD" | Child | (Illegal) |

As the table suggests, the combination of an `OriginalTransactionKey` and setting `StartRecurrent` to true is illegal, or, in best case, meaningless.

How would you model the rules laid out in the above table? In languages like C#, it's difficult, but in F# it's easy.

### C# attempts

Most C# developers would, I think, attempt to model a payment transaction with a class.
If they aim for [poka-yoke design](http://blog.ploeh.dk/2011/05/24/Poka-yokeDesignFromSmelltoFragrance), they might come up with a design like this:

```
public class PaymentType
{
    public PaymentType(bool startRecurrent)
    {
        this.StartRecurrent = startRecurrent;
    }
 
    public PaymentType(string originalTransactionKey)
    {
        if (originalTransactionKey == null)
            throw new ArgumentNullException(nameof(originalTransactionKey));
 
        this.StartRecurrent = false;
        this.OriginalTransactionKey = originalTransactionKey;
    }
 
    public bool StartRecurrent { private set; get; }
 
    public string OriginalTransactionKey { private set; get; }
}
```

This goes a fair way towards making [illegal states unrepresentable](http://fsharpforfunandprofit.com/posts/designing-with-types-making-illegal-states-unrepresentable),
but it doesn't communicate to a fellow programmer how it should be used.

Code that uses instances of this `PaymentType` class could attempt to read the `OriginalTransactionKey`, which, depending on the type of payment, could return null.
That sort of design leads to [defensive coding](http://blog.ploeh.dk/2013/07/08/defensive-coding).

Other people might attempt to solve the problem by designing a class hierarchy:

![A hypothetical payment class hierarchy, showing a Payment base class, and three derived classes: IndividualPayment, ParentPayment, and ChildPayment.](hypothetical-payment-class-hierarchy.png)

(A variation on this design is to define an `IPayment` interface, and three concrete classes that implement that interface.)


This design trades better protection of invariants for violations of the [Liskov Substitution Principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle).
Clients will have to (attempt to) downcast to subtypes in order to access all relevant data (particularly `OriginalTransactionKey`).

For completeness sake, I can think of at least one other option with significantly different trade-offs: applying the [Visitor design pattern](https://en.wikipedia.org/wiki/Visitor_pattern).
This is, however, quite a complex solution, and most people will find the disadvantages greater than the benefits.

Is it such a big deal, then? After all, it's only a single data value (`OriginalTransactionKey`) that may or may not be there.
Surely, most programmers will be able to deal with that.

This may be true in this isolated case, but keep in mind that this is only a motivating example.
In many other situations, the domain you're trying to model is much more intricate, with many more exceptions to general rules.
The more dimensions you add, the more difficult it becomes to reason about the code.

### F# model

F#, on the other hand, makes dealing with such problems so simple that it's almost anticlimactic.
The reason is that F#'s type system enables you to model *alternatives* of data, in addition to the combinations of data that C# (or Java) enables.
Such alternatives are called [discriminated unions](https://docs.microsoft.com/en-us/dotnet/articles/fsharp/language-reference/discriminated-unions).

In the code base I'm currently developing, I model the various payment types like this:

```
type PaymentService = { Name : string; Action : string }
 
type PaymentType =
| Individual of PaymentService
| Parent of PaymentService
| Child of originalTransactionKey : string * paymentService : PaymentService
```

Here, `PaymentService` is a record type with some data about the payment (e.g. which credit card to use).

Even if you're not used to reading F# code, you can see three alternatives outlined on each of the three lines of code that start with a vertical bar (`|`).
The `PaymentType` type has exactly three 'subtypes' (they're called cases, though).
The illegal state of a non-null `OriginalTransactionKey` combined with `StartRecurrent` value of `true` is not possible. It can't be compiled.

Not only that, but all clients given a `PaymentType` value *must* deal with all three cases (or the compiler will issue a warning).
Here's one example where our code is creating the JSON document to send to the payment service:

```
let name, action, startRecurrent, transaction =
    match req.PaymentType with
    | Individual { Name = name; Action = action } ->
        name, action, false, None
    | Parent { Name = name; Action = action } -> name, action, true, None
    | Child (transactionKey, { Name = name; Action = action }) ->
        name, action, false, Some transactionKey
```
        
This code example also extracts name and action from the `PaymentType` value, but the relevant values to be aware of are `startRecurrent` and `transaction`.

* For an individual payment, `startRecurrent` becomes `false` and `transaction` becomes `None `(meaning that the value is missing).
* For a parent payment, `startRecurrent` becomes `true` and `transaction` becomes `None`.
* For a child payment, `startRecurrent` becomes `false` and `transaction` becomes `Some transactionKey`.

Notice that the (parent) `transactionKey` is only available when the payment is a child payment.

The values `startRecurrent` and `transaction` (as well as `name` and `action`) are then used to create a JSON document.
I'm not showing that part of the code here, since there's actually a lot going on in the real code base, and it's not related to how to model the domain.
Imagine that these values are passed to a constructor.

This is a real-world example that, I hope, demonstrates why I prefer F# over C# for domain modelling.
The type system enables me to model alternatives as well as combinations of data, and thereby making *illegal states unrepresentable* - all in only a few lines of code.

## Summary

Classes, in languages like C# and Java, enable you to model *combinations* of data.
The more fields or properties you add to a class, the more combinations are possible. This is often useful, but sometimes you need to be able to model alternatives,
rather than combinations.

Some languages, like F#, Haskell, OCaml, Elm, Kotlin, and many others, have type systems that give you the power to model both combinations and alternatives.
Such types systems are said to have [algebraic data types](https://en.wikipedia.org/wiki/Algebraic_data_type), but while the word sounds 'mathy',
such types make it much easier to model complex domains.

There are more reasons to love F# than only its algebraic data types, but this is the foremost reason I find it a better language for mainstream development work than C#.

If you want to see a more complex example of modelling with types, a good next step would be the first article in my [Types + Properties = Software](http://blog.ploeh.dk/2016/02/10/types-properties-software) article series.

Finally, I should be careful that I don't oversell the idea of making illegal states unrepresentable.
Algebraic data types give you an extra dimension in which you can model domains, but there are still rules that they can't enforce.
As an example, you can't state that integers must only fall in a certain range (e.g. only positive integers allowed).
There are other type systems, such as [dependent types](https://en.wikipedia.org/wiki/Dependent_type), that give you even more power to embed domain rules into types,
but as far as I know, there are no type systems that can fully model all rules as types. You'll still have to write some code as well.
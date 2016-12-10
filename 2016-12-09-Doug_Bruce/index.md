

# Build Yourself a Robo-Advisor in F#. Part I : Domain Modelling #

*All text and code copyright (c) 2016 by Doug Bruce. Used with permission.*

*Original post dated 2016-12-09 available at http://dougbruce.blogspot.com.by/2016/12/build-yourself-robo-advisor-in-f-part-i.html*

**By Doug Bruce**

*This article is based on the first part of the 'Build yourself a Robo-Advisor in F#' workshop, as presented at the [2016 F# Progressive F# Tutorials](https://skillsmatter.com/conferences/7431-progressive-f-sharp-tutorials-2016) conference in London.*

## Introduction

Most of us aren't saving enough for our future. This isn't something that's going to go away - we're living longer and many of us are underestimating how much money
we need for a comfortable retirement. How can you solve this problem? By harnessing the power of F# to deliver clear and easy-to-understand advice and
recommendations that takes you on a journey from confused to confident.

Over the last couple of years, robo-advisors have emerged as a platform for automating this advice as one part of the Fintech revolution.
In this short series of posts we will build a web-based robo advisor that will show you how much money you might have in retirement.

Along the way you will discover how to:

* Understand a domain and model it with the F# type system
* Leverage F#'s data capabilities and make a first program that can chart our projected savings
* Take it to the next level by building a web-based robo-advisor using one of F#'s fantastic open-source web app frameworks

## Part 1 - Domain Modelling

A good first step towards building a new piece of software is to model the domain: what exactly is it we're getting the software to do?
What are the real-life equivalents of the code we want to write?

### What is our domain

We boil our domain down to one example question:

> How much money will I have when I retire?

More generally, we want to take **someone** and calculate **how much** their pension pot will be **worth** when they retire.

The bold sections give us our basic domain:

* People
* Money
* Time


### How can F# help us model it?


**Algebraic Data Types**

[Discriminated Unions](https://fsharpforfunandprofit.com/posts/discriminated-unions/) allow us to express real-world concepts almost directly in code.

Let's take a really simple one with only two options, the boolean. How to we write this in F#?

```fsharp
type Bool = True | False
```

What if we want to say that something can either have a value or not?

```fsharp
type Option<'a> = 
| Some of 'a
None
```

What about if something can fail?

```fsharp
type Result<'TSuccess, 'TFailure> = 
| Succcess of 'TSuccess
| Failure of 'TFailure
```

And what if we want to distinguish one type of string from another?

```fsharp
type EmailAddress = EmailAddress of string
```

That's all pretty useful! We can combine them with [Records](https://fsharpforfunandprofit.com/posts/records/).
Immutable records have a concise syntax and structural equality which make them really powerful for modelling. Here's an example:

```fsharp
type Car = {
  Engine : Engine
  Wheels : Wheel list
  Seats : ... }
```

**Pattern matching**

Once we have our unions, we have a type-safe `switch` alternative in the form of [pattern matching](https://docs.microsoft.com/en-us/dotnet/articles/fsharp/language-reference/pattern-matching).
These give us a one-step match & decompose workflow, and for extra benefit the compiler catches non-exhaustive match so that we are warned if we've added a case and not properly handled it.

Here's an example using a `Result`:

```fsharp
let handleResult = function
| Success result -> "Woop! It all worked, the answer is: " + result
| Failure error -> "Uh-oh. Something bad happened: " + error
```

**Units of measure**

The question we want to answer here is:

> How do you represent a currency in code?

One way to do so is with [Units of Measure](https://fsharpforfunandprofit.com/posts/units-of-measure/).
With these we can make sure something has the right unit, as well as creating derives measures ( e.g `kg m / s^2`)

We can also reduce errors in conversions & calculations. How bad can this really be? Take a look [here](https://www.wired.com/2010/11/1110mars-climate-observer-report/)!

### What did we come up with?

Using what we've just discussed, here's a minimal domain model for figuring out how much money someone is likely to have in retirement.

```fsharp
module Domain =
  open System 

  type Gender = Male | Female

  type EmailAddress = EmailAddress of string

  type Person = { 
      Name : string
      Gender : Gender
      DateOfBirth : DateTime
      EmailAddress : EmailAddress option }

  type ContributionFrequency = Monthly | Annual

  type ContributionSchedule<[<Measure>] 'ccy> = {
      Amount : decimal<'ccy>
      Frequency : ContributionFrequency }

  type PensionPot<[<Measure>] 'ccy> = { 
      Value : decimal<'ccy>; 
      ValuationDate : DateTime; 
      ContributionSchedule: ContributionSchedule<'ccy> }

  [<Measure>] type GBP
  [<Measure>] type USD
  [<Measure>] type EUR

  let valuePensionPot<[<Measure>] 'ccy> (pensionPot : PensionPot<'ccy>) (valuationDate: DateTime) = 
     match pensionPot.ContributionSchedule.Frequency with 
     | Monthly ->
        let monthsInFuture = (valuationDate.Year - pensionPot.ValuationDate.Year) * 12 + valuationDate.Month - pensionPot.ValuationDate.Month
        pensionPot.Value + decimal monthsInFuture * pensionPot.ContributionSchedule.Amount
     | Annual ->
        let yearsInFuture = valuationDate.Year - pensionPot.ValuationDate.Year
        pensionPot.Value + decimal yearsInFuture * pensionPot.ContributionSchedule.Amount
```

I think this code is pretty short and easy to understand given the complexity of the domain. We've used a wide variety of features that helped us along the way.


## Useful Links


Talks

* Scott Wlaschin NDC: https://vimeo.com/97507575
* Tomas Petricek Channel 9: https://www.youtube.com/watch?v=Sa6bntNslvY
* Mark Seeman NDC: https://vimeo.com/131631483

Blogs

* http://fsharpforfunandprofit.com/ddd/
* http://tomasp.net/blog/type-first-development.aspx/
* http://gorodinski.com/blog/2013/02/17/domain-driven-design-with-fsharp-and-eventstore/

## Wrapping up

In this part you've seen how we take take a real-life domain and come up with a concise and accurate model for it using F#.

We've got some great language features to model domains in F#! If you are thinking about how to represent things in code, the ideas in this post are a great starting point.


## Next Time

Charts and data! You'll see how to take the domain we've just created and chart it in a Desktop app with [FSharp.Charting](https://fslab.org/FSharp.Charting/).
We'll also look at accessing data via [type providers](https://docs.microsoft.com/en-us/dotnet/articles/fsharp/tutorials/type-providers/).
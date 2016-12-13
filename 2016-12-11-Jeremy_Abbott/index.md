
# Merry Monad Xmas #

*All text and code copyright (c) 2016 by Jeremy Abbott. Used with permission.*

*Original post dated 2016-12-11 available at https://jeremyabbott.github.io/blog/2016/12-03-merry/*

**By Jeremy Abbott**

I've been playing with F# and functional programming (FP) as a hobby for a few years now. For the past month I've had the privilege of working with a mentor from the F# Foundation. If you're not a member of the F# Foundation, stop reading right now and [go join](http://foundation.fsharp.org/join).

One (of many!) aspects of functional programming that's intimidated me has been the concept of monads. For some reason I've felt that the full power of FP would become clear if I could understand this concept. I have a bad habit of shying away from concepts that sound too mathematically abstract, and I always felt that monads fell into that category. It's a personal failing I'm trying to get better at. With that being said, when I got paired up with [Clément](https://twitter.com/cboudereau), we started working on monads immediately. For my contribution the 2016 F# Advent Calendar I hope to explain how I understand monads in the hope that it might help you too!

## Setting Up The Story

Santa needs to deliver presents to the millions of good children that made the nice list. Those naughty children that didn't make the cut will be receiving a lump of coal.

```fsharp
open System

type Gift =
| Wants of string
| Coal

type Person = { Name: string; SharedToys: bool; IsANastyWoman: bool; Wants: string }

let giftIf behavior person =
    let gift = if behavior then Wants person.Wants else Coal
    person.Name, gift
let sharedToys person = giftIf person.SharedToys person
let wasANastyWoman person = giftIf person.IsANastyWoman person

let determineGift person =
    match sharedToys person with
    | n, Coal -> n, Coal // fail fast
    | _, _ ->
        match wasANastyWoman person with
        | n, Coal -> n, Coal // fail again
        | n, Wants s -> n, Wants s 

let naughtyOrNice person =
    person |> determineGift

naughtyOrNice { Name = "Jeremy"; // Jeremy gets a redo on 2016
                IsANastyWoman = true;
                SharedToys = true; 
                Wants = "For 2016 not to have happenend" }

naughtyOrNice { Name = "Donald"; // Donald gets coal
                IsANastyWoman = false;
                SharedToys = false;
                Wants = "To maintain systemic discrimination and privilege." }
```
                
The preceding code defines a record type with a `Name` property, and the criteria Santa is using this year to determine if someone is naughty or nice. Next we define some functions to validate each person, and then ultimately determine if they are naughty or nice.

The functions `sharedToys` and `wasANastyWoman` return tuples of the type `string * Gift` where the Gift's case is based on whether or not they were a decent person. If they weren't a decent person they get a nice lump of coal for Christmas.

The function `determineGift` determines whether the person gets Coal or the gift they requested. Our problem starts here:

```fsharp
// this is not ideal
let determineGift person =
    match sharedToys person with
    | n, Coal -> n, Coal // fail fast
    | _, _ ->
        match wasANastyWoman person with
        | n, Coal -> n, Coal // fail again
        | n, Wants s -> n, Wants s  // what if we need to add more validation rules?
```
        
As we add more rules for determining gifts we have to keep extending this function. "Arrow code" like this is a code smell. This kind of code isn't very idiomatic in functional programming either. Yes, we have small functions that do one thing, but we don't have a nice way of composing them together.

To make this a little more idiomatic we could change `Person` like this:

```fsharp
type Person' = { Name: string; SharedToys: bool; IsANastyWoman: bool; Gift: Gift }
```

And then change the validation functions to behave like this:

```fsharp
type Person' = { Name: string; SharedToys: bool; IsANastyWoman: bool; Gift: Gift }

let giftIf' behavior (person: Person') =
    if behavior then person
    else { person with Gift = Coal }

let sharedToys' person = giftIf' person.SharedToys person

let wasANastyWoman' person = giftIf' person.IsANastyWoman person

let determineGift' = sharedToys' >> wasANastyWoman'
```

Now we can add new functions to `determineGift` via composition. Unfortunately it's not clear what kind of gift a person gets without inspecting the Person' record. We would have to match on the shape of the record type if we wanted to filter on people who were naughty or nice.

## Using a Result Type

In the previous iteration we recognized that it would be nice if the inputs and outputs to our validation functions were more uniform because it gave us the flexibility to compose our functions. There is a problem with this latest approach though: All of the validation functions are executed, which means we can't quick return if any of them fail.

The previous iteration can be improved upon with a couple more adjustments. The previous example mentioned that the caller can't tell if someone was "naughty" or "nice" without inspecting properties on the result. It would be nice if the caller could tell from the result type if the person belonged on the good list or the bad list.

## Enter the Result Type

```fsharp
type Result<'t> =
| Nice of 't
| Naughty of 't
```

If the `Person` was nice, the validation functions should return a result of `Nice`, and if they were naughty, the functions should return a result of `Naughty`.


```fsharp
let giftIf'' behavior (person: Person') =
    if behavior then Nice person
    else Naughty { person with Gift = Coal }

let sharedToys'' person = giftIf'' person.SharedToys person

let wasANastyWoman'' person = giftIf'' person.IsANastyWoman person

let determineGift'' person = 
    match sharedToys'' person with
    | Naughty p -> Naughty p
    | Nice p ->
        match wasANastyWoman'' person with
        | Naughty p -> Naughty p
        | Nice p -> Nice p
```
        
Awesome! The output of the validation functions is now easy to match on. But, the original problem of arrow code and an inability to chain functions is back. It would be nice if the functions were chainable, and if the functions would fail fast meaning that if validation if one function fails, the logic of subsequent functions isn't executed.

```fsharp
// function that accepts a (person -> Result) and a Result
let failFast nOrFF nOrF =
    match nOrF with
    | Naughty p -> Naughty p
    | Nice p -> nOrFF p
```
    
The function `failFast` accepts any function with the signature `('a -> Result<'a>)` and returns a function with a signature of `(Result<'a> -> Result<'a>)`. Said another way it's a function that accepts `('a -> Result<'a>)` and a `Result<'a>` and returns `Result<'a>`.

Using `failFast` we can rewrite `determineGift` as follows:

```fsharp
let determineGiftRedux person =
    person
    |> sharedToys''
    |> failFast wasANastyWoman''
    
determineGiftRedux { Name = "Donald"; IsANastyWoman = false; SharedToys = false; Gift = Wants "To maintain systemic discrimination and privilege." }
```

Using the `failFast` function `determineGift` can be rewritten using a the pipe operator in something that is idiomatically F#.

Another key piece of the `failFast` function is that once we determine that a `Person'` is `Naughty` subsequent validation functions no longer get called. This can be really important if a validation function performs an expensive operation because it gets skipped if we already have enough information to make a determination.

We can make `failFast` even prettier:

```fsharp
let (>>=) nOrF nOrFF = failFast nOrFF nOrF

let determineGiftRedux' person =
   person
   |> sharedToys'' 
   >>= wasANastyWoman''

determineGiftRedux' { Name = "Donald"; IsANastyWoman = false; SharedToys = false; Gift = Wants "To maintain systemic discrimination and privilege." }
```

A few things are happening here:

1. A custom operator is created for `failFast`. It accepts a `Result` type on the left side of the operator, and a function of type `'a -> Result<'a>` on the right side.
1. A person is piped to the first validation function using the built-in pipe operator. This returns a `Result<Person'>` which can be passed to the new custom operator.
1. The next validation function is passed as the second argument to the `>>=` custom operator.
1. Additional validation functions can be included in the pipeline using `>>=` at this point, and they will only get called if the `Result` matches the `Nice` case.

## failFast has Another Name

The `failFast` function has another, more idiomatic name: `bind` or `flatMap`. The name `failFast` makes sense for the type we're working with, but any type can implement this function and the underlying behavior could be very different. The general signature for this function is `('a -> M<'a>) -> M<'a> -> M<'b>`. This function's purpose is to take a function with a signature of `'a -> M<'a>` and convert it to a function with the signature `M<'a> -> M<'b>`. For me this was (is?) hard to grasp at first because as much as I delve into function programming I forget about key features of the style like currying and partial application.

When only the first argument is passed to bind via partial application, a new function is returned, and that function has the signature `M<'a> -> M<'b>`. That new function allows the caller to chain functions, threading `M<'a>` through as the argument. Neat!

## Introducing Map

The `Result` type has turned out to be pretty useful for this exercise. However, how can we use the `Result` type in practice if we have functions that weren't written to work with `Result`? For example, what if a lot of people are asking for a "redo on 2016?" That seems really valid, but isn't the most feasible gift. Santa suggests that they just need a time out with the latest Pokémon game because Pokémon makes everything better.

```fsharp
let needsPokemon p =
    match p with
    | { Gift = Wants "For 2016 not to have happened" } -> { p with Gift = Wants "Pokémon Sun/Moon" }
    | _ -> p
```
    
However, the function `needsPokemon` has the signature `(Person' -> Person')` and our pretty function pipeline only works with `(Result<'t> -> Result<'t>)`. This is where the map function comes in handy:

```fsharp
let map f r =
    match r with
    | Nice n -> Nice (f n) 
    | Naughty n -> Naughty n

let determineGiftRedux'' person =
    person
    |> sharedToys'' 
    >>= wasANastyWoman''
    |> map needsPokemon

determineGiftRedux'' { Name = "Jeremy"; IsANastyWoman = true; SharedToys = true; Gift = Wants "For 2016 not to have happenend" }
(* 
    Result<Person'> = Nice { Name = "Jeremy";
                            SharedToys = true;
                            IsANastyWoman = true;
                            Gift = Wants "Pok�mon Sun/Moon";}
*)
```

Using the `map` function we were able to take a function that wasn't designed to work with our `Result` pipeline and convert it into one that was.

## Spoiler: Result is a Monad

So far this excercise has a allowed us to:

1. Cathartically rip on what has been a disappointing year
1. Help Santa easily manage his naughty or nice lists
1. Learn about bind (flatMap) and map
1. *Use a monadic type without really talking about monads!*

While points 1 and 2 are fun, let's focus on points 3 and 4. Any type that has both a map and a bind function is actually a monadic type. Any type that implements a map function, where that map function follows a few rules, is called a functor. A monad can then be said to be a functor that also implements flatMap.

If you've worked with F# for any period of time, you've probably used a couple monadic types without knowing it. Before learning about monads, I felt like working with the `Option` type was nice for being explicit about what a function should accept or return, but that it was awkward in practice. Who would want to have to constantly match on `Some` and `None`? Well, it turns out that's not really the best way to work with that type:

```fsharp
// signature is string -> Option string
let optionalString s =
    if System.String.IsNullOrEmpty(s) then None
    else Some s

// signature is string -> Option string
let withCharacters s =
    if System.String.IsNullOrWhiteSpace(s) then None
    else Some s

let empty = ""
let notEmpty = "Hello fellow nasty woman"
let whitespace = " "
let oEmpty = optionalString empty // None
let oNotEmpty = optionalString notEmpty // Some string
let oWhiteSpace = withCharacters whitespace // None

let shouldBeNone =
    whitespace
    |> optionalString
    |> Option.bind withCharacters // checkout the usage of bind!

let shouldBeSome =
    notEmpty
    |> optionalString
    |> Option.bind withCharacters // checkout the usage of bind! 
```
    
Obviously this example is pretty contrived, but the point is still valid: using `Option.bind` made working with functions that return `Option` in a pipeline much clearer.

## Happy Holidays!

I hope that this simple example using a slightly modified `Result` example helped you start to grasp how monadic types work, and how they can be helpful. There is a wealth of knowledge on monads out there, and these are the resources that helped me out:

* [Railway Oriented Programming](http://fsharpforfunandprofit.com/rop/)
* [Understanding Map](http://sidburn.github.io/blog/2016/03/27/understanding-map)
* [Fun fun function: Monads](https://www.youtube.com/watch?v=zFO1cRr5-qY)
* [Understanding Monads: Real World F#](https://medium.com/real-world-fsharp/understanding-monads-db30eeadf2bf#.toimrjpo8)
* [F# Foundation](http://fsharp.org/) (specifically the mentor program. Thank you Clément!)

## Last Words

If an example here is unclear, or worse, is innaccurate please submit a pull request. I write these posts to help myself as much as being a source of information for others and that doesn't work if the information is wrong.

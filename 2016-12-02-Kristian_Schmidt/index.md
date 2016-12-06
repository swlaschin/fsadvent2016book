
# F# in Production #

*All text and code copyright (c) 2016 by Kristian Schmidt. Used with permission.*

*Original post dated 2016-12-02 available at http://www.kreutz.us/2016/12/02/f-in-production/*

**By Kristian Schmidt**


What the most efficient way to get the F# community into the christmas spirit? Someone using F# in production, surely. So today I'm going to give a little intro to our latest project, and reflect a bit on the challenges we faced.

## Intro

I work at [PFA Pension](https://www.pfa.dk/) in the actuarial modelling department, where our primary task is to calculate the total liabilities for the company's insurance policies. The liability is the amount of money we need to set aside to be sure that we're able to pay our obligations to our customers.

I have [previously spoken](http://www.kreutz.us/2016/01/20/slides-from-my-talk-at-mf-k/) about how we've used F# to calculate the value of individual contracts. This is the extension of that work, in which we consider entire policies consisting of multiple contracts. So where calculating the value of one contract was math heavy, this project's focus is more on implementing business rules and taming the complexity of these, along with fleshing out the structure of the problem domain in types.

I won't be getting into too many specifics, but instead I'll focus on self contained bits I think the community will find interesting.

## The Existing Solution

Pension companies having to calculate liabilities is nothing new, and we already had an existing solution in place. This was written in the SAS statistical programming language. For those of you that, thankfully, don't know about it, SAS is a row-based data-processing language, where you write a function (a so called data-step) to process each row of a table. The main abstraction is a macro, which is just like a function, except it's just directly inserted into your code. So you have to be careful not to use the same variable names from your macros in your other code. Oh, and did I mention that all variables are global?

People often shun the big rewrite, but in our case, we were at a place where no one wanted to change the existing code in fear of what might happen. We were also using a language that was generating additional complexity instead of limiting it. So, for me at least, it was a no brainer.

## Getting Data

To compute anything we need to get some data into an appropriate model. Here we make extensive use of [Railway-oriented-programming](http://fsharpforfunandprofit.com/rop/),
which enables us to streamline the error prone process of parsing data.

Let's look at a little example.


```fsharp
type DataRecord = { PropX : X; PropY : Y }

type Error = | XError of string | YError of string

type Result<'a> = | OK of 'a | Failure of Error

let readX : X = function
                | "1" -> OK X1
                | "2" -> OK X2
                | s   -> Failure (XError <| sprintf "Unknown X type %s" s)

let readY = ...

let example xStr yStr =
    data {
        let! x = readX xStr
        let! y = readY yStr

        return { PropX = x; PropY = y }
    }
```
    
Here we're able to contain the error handling logic in individual, small, functions and then compose them with our `data` computation expression,
which just makes sure none of the `let!` bindings return `Failure`, returning the first failure if there are any.

Not only did this make for a relatively pleasant experience doing data parsing. This also gave us an incremental view of how much of the input
data we were able to parse correctly. Since all of the parsed data will be wrapped in a `Result`, measuring completeness was a simple matter of
counting how many `Results` were `OK` vs `Failure`.

We also took the same approach to handle possible errors in the liability calculations themselves.
For each calculation, we call an external DLL several times, and this might throw an exception. If that happens, the exception is wrapped in the `Failure` case of `Result`,
avoiding crashing the program. The advantage here is that whenever you run the program with a large workload, you can be sure it doesn't crash halfway through,
and any errors that might pop up can be saved and analyzed later.

## Debugging made easy with the REPL

Debugging idiomatic F# isn't always a lot of fun. If you try to step through your program, it jumps back and forth between function definitions.
And if you're using custom computation expressions like us, you're in for an incomprehensible ride through your code.

After a certain amount of time, you iron out all of the show stopping bugs, and debugging becomes a matter of understanding why the model outputs the numbers it does.
To aid with this, what we've done is define a helper script that can load up a single policy and then run various debugging functions on it.
Here [Deedle](http://bluemountaincapital.github.io/Deedle/) plays a big part, since we not only use its basic data frame functionality,
but we also [convert our existing types to be displayed as a data frame](http://www.kreutz.us/2016/05/26/f-interactive-pretty-printing-with-deedle/).

Debugging in this way with the REPL is a huge timer saver, because all the data you need will be right there in your FSI session. And if you need a question answered, you just write a function to compute it and run it immediately! If you consider the alternative, which would be to mangle the program with Console.WriteLine and chuck that into Excel for analysis, this approach is faster, easier and more flexible.

## Testing

Given that this is a re-write of an existing model, we can do a very thorough test wrt. the final results. These should simply match the output of the old model.

As for unit testing, I've found myself just not writing that many. Partly because of the type system and immutability by default, but also because of how many of the business rules we implement are structured.
A lot of cases are just: "if case A then multiply value by 5%, else multiply value by 3%".
So naturally, for a given function, I can write a test that tests if this holds up, but what have I really achieved here? I'm just implementing the same logic twice. If do things correctly, the test merely serves as a lock on existing results, and any legal change in the implementation will have me changing the test too. If I've misunderstood the rule, the test won't help me catch the error because I'm also implementing the wrong test.

It's not like a sorting algorithm, where the test for correctness is vastly different from the algorithm, and there are many legal implementations which will pass tests.

Comments are greatly appreciated here, as it's a question I've been struggling with. Right now we rely mostly on a full model test, which can't be run on our workstations, and therefore don't get run as often as the unit tests.

##Performance

Performance matters. Even though you're not doing real time processing of incoming requests, a better performing program is just easier to work with. In our case, being able to run the model 50 times instead of say, 10 times, a day makes us able to answer more questions about its behaviour, and provide more detailed answers to those questions.

First a little bit about the type of workload we have. We have a lot (can't share specifics, unfortunately) of insurance policies that need a few different numbers calculated for each of them. The policies are completely independent, so the problem becomes an embarassingly parallel one. We run the model on a 64 core machine with 128 GB of RAM, so we're able to keep a lot of data in memory, if need be.

## (Im)mutability

Pretty much all data structures are immutable, except for a few key ones. Our main unit of computation is a `Cashflow`,
which is just a labeled float array with a length somewhere between 1 and 120. We allocate on the order of XX millions of these every time we run the program.

While a standard array is inherently mutable, we have defined the standard operators (+, -, *, /) for a `Cashflow` such that they create a new `Cashflow`
with a new underlying array each time. So we get the benefits of immutability at the price of the developers having to know not to manually manipulate the underlying arrays. It's not the guarantee you'd get from using a linked list, but in our case the trade off makes sense.

There's no doubt we could achieve better performance by just using a few arrays per policy and manipulating them over and over again. But when you factor in the added development time for the inevitable bug hunting and having to write a test suite to test all sorts of weird interactions, we deemed it wasn't worth it. Now that we've started using the model in production, I'm happy to say that it ended up being fast enough for our use.

## ToString()

Our output is just a csv file with a bunch of numbers and identifiers in them. Some of these identifiers are just the `ToString` method
of a discriminated union without any data attached. Starting out, I thought I'd be really smart and implemented `ToString()` like this:

```fsharp
type Example =
    | A | B
    static member this.ToString() = sprintf "%A" this
```    
    
The good part is that if you expand your DU here, you don't have to touch the `ToString` method.
The bad part is that this ended up being a severe performance bottleneck in several cases!
Turns out this isn't so fast if you want to call the method millions of times. The fix was simple:

```fsharp
type Example =
    | A | B

    static member this.ToString() =
        match this with
        | A -> "A"
        | B -> "B"
```
        
Now you have to change the method if you expand the DU, but even if you forget compiler warnings will remind you of the missing pattern match.
This change fixed all of our ToString-related performance issues.

## GC Stats

Just for fun I thought I'd give you some numbers. Below you'll see a table of the GC stats that [PerfView](https://github.com/Microsoft/perfview) generates.
These stats were generated running the program on a 64 core machine with `CLR Startup Flags: CONCURRENT_GC, SERVER_GC`.
Only the portion of the code that calculates stuff was profiled. The generation of output was skipped.

| GC Index | Pause Start | Trigger Reason | Gen | Pause MSec | % Pause Time | Gen0 Alloc MB | Gen0 Alloc Rate MB/sec |
|---|---|---|---|---|---|---|---|
| 2 | 15.681,315 | AllocSmall | 0N | 5.204,196 | 50,4 | 682,229 | 133,36 |
| 3 | 31.311,055 | AllocSmall | 2B | 495,062 | 29,3 | 6.339,091 | 2.200,44 |
| 4 | 31.350,123 | AllocSmall | 1N | 5.687,250 | 35,2 | 3.776,353 | 360,86 |
| 5 | 41.291,427 | AllocSmall | 0F | 898,578 | 17,3 | 5.953,185 | 1.381,22 |
| 6 | 46.439,654 | AllocSmall | 1N | 1.225,027 | 47,2 | 6.520,651 | 4.763,03 |
| 7 | 54.761,537 | AllocSmall | 0N | 917,653 | 11,4 | 9.696,989 | 1.366,35 |
| 8 | 59.798,991 | AllocSmall | 0N | 880,756 | 17,6 | 10.376,745 | 2.518,66 |
| 9 | 65.247,703 | AllocSmall | 2B | 1.776,603 | 28,7 | 2.990,233 | 2.402,10 |
| 10 | 65.260,024 | AllocSmall | 0N | 945,024 | 17,1 | 10.985,220 | 2.398,28 | 
| 11 | 74.186,778 | AllocSmall | 1F | 2.131,579 | 21,1 | 1.505,170 | 188,39 |
| 12 | 80.610,316 | AllocSmall | 0N | 1.041,057 | 25,5 | 12.121,152 | 3.977,32 |
| 13 | 85.577,245 | AllocSmall | 0N | 1.007,499 | 20,4 | 12.253,635 | 3.121,14 |
| 14 | 90.949,361 | AllocSmall | 0N | 986,747 | 18,4 | 12.584,146 | 2.883,12 |
| 15 | 96.015,221 | AllocSmall | 1N | 2.276,321 | 35,8 | 12.645,542 | 3.099,95 |
| 16 | 102.378,983 | AllocSmall | 0N | 1.088,974 | 21,0 | 12.703,983 | 3.107,93 |
| 17 | 106.666,817 | AllocSmall | 0N | 832,306 | 20,6 | 11.686,306 | 3.653,11 |
| 18 | 112.030,120 | AllocSmall | 0N | 1.000,396 | 18,1 | 12.510,029 | 2.760,88 |
| 19 | 117.049,092 | AllocSmall | 1N | 2.130,006 | 34,6 | 12.518,743 | 3.115,09 |
| 20 | 123.000,183 | AllocSmall | 0N | 1.028,213 | 21,2 | 12.470,023 | 3.263,34 |
| 21 | 128.056,713 | AllocSmall | 0N | 1.064,758 | 20,9 | 12.588,864 | 3.124,97 |
| 22 | 133.372,027 | AllocSmall | 0N | 1.060,356 | 20,0 | 12.756,741 | 3.001,08 |
| 24 | 138.586,971 | AllocSmall | 0N | 1.115,656 | 21,2 | 13.464,913 | 3.240,87 |

```
Total Allocs : 212.300,084 MB
Total GC Pause: 34.794,0 msec
% Time paused for Garbage Collection: 28,1%
Max GC Heap Size: 26.251,720 MB
```

That's a lot of allocations! 3 GB per second! The program ran for a bit over 2 minutes and allocated over 200 GB. The max heap size was 26 GB, so the vast majority of these allocs didn't make it past gen 0. Normally we say gen 0 GCs are supposed to be fast, but I guess you can put pressure on those too, after all?

On one hand, the high performance guy in me wants to remove a ton of these allocations at the expense of code safety and development time, but right now we're at a point where this is just good enough. The model is several times faster than our old solution, so at the moment everyone is praising performance. At the same time the development was a smooth experience, where most of the debugging time was spent figuring out what the old SAS code was really doing.

## Summary

In closing, I just want to touch a bit on what I think made this project a success. It wasn't profunctor optics. It was just the continued application of better types (DUs, records), immutability and better error handling (railway oriented programming). This allowed us to focus on the business logic instead of the technical stuff.

And finally, a shout out to FAKE, Paket & F# Power Tools (we're not on Code/Ionide ...yet), which makes our lives easier every day.

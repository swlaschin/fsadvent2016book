
# Parsing text in F# #

*All text and code copyright (c) 2016 by Erik Schierboom. Used with permission.*

*Original post dated 2016-12-10 available at http://www.erikschierboom.com/2016/12/10/parsing-text-in-fsharp/*

**By Erik Schierboom**

To hone my programming skills, I’ve been using [exercism.io](http://exercism.io/) a lot.
With Exercism, you can work on problems in over [30 different languages](http://exercism.io/languages).
In this blog post, we’ll be solving exercism’s [wordy](http://exercism.io/exercises/fsharp/wordy/readme) problem, which focuses on text-parsing.
We’ll be using F# as our language of choice and use various approaches to solve the problem. In the process, we’ll do lots of refactoring.

## Setting up Exercism

Setting up Exercism is simple. It involves three steps:

1. Create an account by [logging in](http://exercism.io/login) to Exercism with your GitHub credentials.
1. Install the Exercism [command-line client](http://exercism.io/cli), which is available for Windows, MacOS and Linux.
1. Configure the command-line client with your account’s [API key](http://exercism.io/account/key):

```
exercism configure --key=<your API key>
```

Once these steps have been completed, you can use the CLI to fetch and submit problems. If you’d like more detailed instructions on how to setup Exercism, please follow their [detailed tutorial](http://exercism.io/how-it-works/newbie).

## Problem 1: Wordy

To start working on the wordy problem with F#, we use this CLI command:

```
exercism fetch fsharp wordy
```

This command will fetch the wordy exercise’s [README](http://exercism.io/exercises/fsharp/wordy/readme) and [test suite](http://exercism.io/exercises/fsharp/wordy/test-suite), and write them to the file system:

```
Not Submitted:            1 problem
fsharp (Wordy) /Users/erikschierboom/exercism/fsharp/wordy

New:                      1 problem
fsharp (Wordy) /Users/erikschierboom/exercism/fsharp/wordy

unchanged: 0, updated: 0, new: 1 
```

If we examine the contents of the `wordy` directory, we’ll find two files:

* `README.md`: a textual description of the exercise.
* `WordyTest.fs`: the test suite, to verify the correctness of the implementation.

As said, the `README` file describes the problem we should solve:

> Write a program that takes a word problem and returns the answer as an integer.

It also lists some sample word problems, such as:

> What is 5 plus 13?

Now that we know the problem to solve, let’s try to solve it. We’ll go at it one test at a time, slowly building up a correct
implementation that passes all tests (which are defined in the `WordyTest.fs` test suite). In the process, we’ll do lots of refactoring, to try to improve the quality of our code. Let’s start!

### Test 1: single digit numbers

The first test is defined as followed:

```fsharp
[<Test>]
let ``Can parse and solve addition problems`` () =
    Assert.That(solve "What is 1 plus 1?", Is.EqualTo(Some 2))
```
    
From this code we can infer that we need to define a `solve` function that takes a string parameter (the equation) and returns an `int option` value (the equation’s result).

Let’s implement a very simple equation solver that removes all non-digit characters and then parses the two remaining digits to calculate the sum:

```fsharp
let solve (question: string) =
    let equation = 
        question
            .Replace("What is ", "")
            .Replace(" plus ", "")
            .Replace("?", "")
    let left  = Convert.ToInt32(equation.Substring(0, 1))
    let right = Convert.ToInt32(equation.Substring(1, 1))
    Some (left + right)
```
    
If we run the test with this code, it passes.

Note that we aligned the `Convert.ToInt` calls. Aligning these statements has two advantages:

1. As the statements are so alike, they give users a clue that they probably have similar functionality.
1. It helps us compare the statements. With this alignment, it becomes easy to see which parts of the strings they convert.

**Refactoring: simplify integer conversion**

Our first refactoring is to use the `int` function instead of `Convert.ToInt32`:

```fsharp
let solve (question: string) =
    let equation = 
        question
            .Replace("What is ", "")
            .Replace(" plus ", "")
            .Replace("?", "")
    let left  = int <| equation.Substring(0, 1)
    let right = int <| equation.Substring(1, 1)
    Some (left + right)
```
    
A quick check verifies that our code still passes the test.

We can also replace the `Substring` calls with array slice syntax:

```fsharp
let solve (question: string) =
    let equation = 
        question
            .Replace("What is ", "")
            .Replace(" plus ", "")
            .Replace("?", "")
    let left  = int equation.[0..0]
    let right = int equation.[1..1]
    Some (left + right)
```
    
**Refactoring: extract helper functions**

Next, we can make the `solve` function a bit easier to read by extracting some helper functions:

```fsharp
let parseEquation (question: string) = 
    question
        .Replace("What is ", "")
        .Replace(" plus ", "")
        .Replace("?", "")

let parseLeft  (equation: string) = int equation.[0..0]
let parseRight (equation: string) = int equation.[1..1]

let solve (question: string) =
    let equation = parseEquation question
    let left  = parseLeft equation
    let right = parseRight equation
    Some (left + right)
```
    
The refactored code looks cleaner to me, and it still passes the test! On to the second test.

### Test 2: double digit numbers

In the second test, we have to parse double digit numbers:

```fsharp
let ``Can add double digit numbers`` () =
    Assert.That(solve "What is 53 plus 2?", Is.EqualTo(Some 55))
```
    
If we run this test, it fails. This is due to us hard-coding the `int` parsing to single digits. To support double digit numbers, we’ll rewrite our code to use regular expressions:

```fsharp
let solve (question: string) =
    let pattern = "What is (\d+) plus (\d+)\?"
    let matched = Regex.Match(question, pattern)
    let left  = int matched.Groups.[1].Value
    let right = int matched.Groups.[2].Value
    Some (left + right)
```
    
This time, the double digit test passes. Although regular expressions can easily become complicated and hard to read, ours is quite simple and understandable. In this case, I feel that using a regular expression does improve the code’s readability.

Note that we assume the regular expression always matches. We’ll defer adding proper error handling until later, when the tests force us to.

**Refactoring: using named expressions**

Although the regular expression is quite simple, I don’t like how we extract the matched digit values using integer indexing.
We can improve this by using `named expressions`. This regular expression feature lets us attach a name to an expression, which we can then use to retrieve the matched value:

```fsharp
let solve (question: string) =
    let pattern = "What is (?<left>\d+) plus (?<right>\d+)\?"
    let matched = Regex.Match(question, pattern)
    let left  = int matched.Groups.["left"].Value
    let right = int matched.Groups.["right"].Value
    Some (left + right)
```
    
You can see that the `(\d+) plus (\d+)` part of the regular expression has been replaced with `(?<left>\d+) plus (?<right>\d+)`.
The `?<left>` and `?<right>` parts specify the names of the expressions. We then use these names to index into the matched groups, instead of using integer indexers.

Re-running our tests verifies that everything still works.

### Test 3: negative numbers

The third test haves us work with negative numbers:

```fsharp
[<Test>]
let ``Can add negative numbers`` () =
    Assert.That(solve "What is -1 plus -10?", Is.EqualTo(Some -11))
```
    
Once again, we have to fix our code to make this test pass. Luckily, the fix is as simple as adding an optional minus (`-`) prefix to the digit matching expressions in our regular expression:

```fsharp
let pattern = "What is (?<left>-?\d+) plus (?<right>-?\d+)\?"
```

### Test 4: large numbers

Our current implementation passes this test, so let’s move on.

### Test 5: subtraction

In test number 5, we have to deal with a new operator, the minus operator:

```fsharp
[<Test>]
let ``Can parse and solve subtraction problems`` () =
    Assert.That(solve "What is 4 minus -12?", Is.EqualTo(Some 16))
```
    
To support the new operator, we first capture it in our regular expression:

```fsharp
let pattern = "What is (?<left>-?\d+) (?<operator>plus|minus) (?<right>-?\d+)\?"
```

The final step is to replace the `Some (left + right`) expression with a match on the operator’s value to determine what value to return:

```fsharp
match matched.Groups.["operator"].Value with 
| "plus"  -> Some (left + right)
| "minus" -> Some (left - right)
```

Running the tests again verifies that they all pass.

**Refactoring: using Active Patterns**

If we look at our current code, it consists of three simple steps:

1. Match the question with our regular expression.
1. Convert the `"left"` and `"right"` groups to `int` values.
1. Based on the matched `"operator"`, combine the left and right values and return it as an `int option` value.

This combination of matching and transforming is so commonplace, that F# created a special construct for it: [Active Patterns](https://fsharpforfunandprofit.com/posts/convenience-active-patterns/).
Let’s create an Active Pattern that will match an equation:

```fsharp
let (|Equation|) question =
    let pattern = "What is (?<left>-?\d+) (?<operator>plus|minus) (?<right>-?\d+)\?"
    let matched = Regex.Match(question, pattern)

    let left  = int matched.Groups.["left"].Value
    let right = int matched.Groups.["right"].Value

    match matched.Groups.["operator"].Value with 
    | "plus"  -> Some (left + right)
    | "minus" -> Some (left - right)
```
    
The main thing to note is that the function’s body is just literally copy-pasted from the `solve` method.
What’s different though, is the function’s identifier. An Active Pattern’s identifier is defined between pipe (`|`) characters, placed within parentheses.

We can use this Active Pattern in our `solve` function as follows:

```fsharp
let solve (question: string) =
    match question with
    | Equation result -> result
```
    
You can see that we can `match` the question `string` using our `Equation` Active Pattern, with the value returned from the Active Pattern being bound to the `result` identifer.

Re-running the tests confirms that everything still works. Looking at our code though, I don’t think the Active Pattern gives us much in terms of readability our functionality, so we’ll revert our code to the previous iteration (which didn’t use an Active Pattern).

### Test 6: multiplication

In test 6, multiplication is added to the list of supported operations:

```fsharp
[<Test>]
let ``Can parse and solve multiplication problems`` () =
    Assert.That(solve "What is -3 multiplied by 25?", Is.EqualTo(Some -75))
```
    
This is trivial to add. We simply add the `"multiplied by"` string to the operator part of our regular expression, and then handle this operator in our `match` clause:

```fsharp
match matched.Groups.["operator"].Value with 
| "plus"          -> Some (left + right)
| "minus"         -> Some (left - right)
| "multiplied by" -> Some (left * right)
```

With that, the test passes.

### Test 7: division

Test 7 adds supports for division:

```fsharp
[<Test>]
let ``Can parse and solve division problems`` () =
    Assert.That(solve "What is 33 divided by -3?", Is.EqualTo(Some -11))
```
    
Once again, this is trivial to add, so let’s skip the implementation details.

### Test 8: add twice

Things start getting more interesting in test 8, where we have to add not once, but twice!

```fsharp
[<Test>]
let ``Can add twice`` () =
    Assert.That(solve "What is 1 plus 1 plus 1?", Is.EqualTo(Some 3))
```
    
So far, we have only considered simple expressions with two integers and one operand. However this time, we start with the leftmost number and then apply two of operations.
The most hacky solution would be to just match an optional second operation:

```fsharp
let pattern = "What is (?<left>-?\d+) (?<operator1>plus|minus|multiplied by|divided by) (?<right1>-?\d+)( (?<operator2>plus|minus|multiplied by|divided by) (?<right2>-?\d+))\?"
```

However, I don’t think this is very readable, nor is it very future-proof (what if we want to support five operations?).
Let’s take a step back to figure out a better approach. Sometimes it helps to make the state transitions explicit, so let’s try that for the `"What is 1 plus 1 plus 1?"` question:

1. Set the current state to 1.
1. Add 1 to the current state (1). Returns 2.
1. Add 1 to the current state (2). Returns 3.

I don’t know about you, but this looks an awful lot like a left fold to me. To use a left fold in our implementation, we first have to ensure that we match the added second operation. Our initial attempt at this might look like this:

```fsharp
let pattern = "What is (?<left>-?\d+)(?<operations> (?<operator>plus|minus|multiplied by|divided by) (?<right>-?\d+))+\?"
```

Here, we define a new capture group named `"operations"` that matches one or more operator/operand combinations. Unfortunately, this did not work.
Let’s try another approach where we split the operations matching part into its own pattern:

```fsharp
let equationPattern = "What is (?<left>-?\d+)(?<operations>.+?)\?"
let operationsPattern = " (?<operator>plus|minus|multiplied by|divided by) (?<right>-?\d+)"
```

Next, we’ll do two matches:

1. We match the equation (without matching the actual operations).
1. We match the operations using the matched `"operations"` part of the equation match:

```fsharp
let equationMatch = Regex.Match(question, equationPattern)
let operationMatches = Regex.Matches(equationMatch.Groups.["operations"].Value, operationsPattern)
```

We can then calculate the equation’s result by folding on the matched operations, with the leftmost integer as the starting state:

```fsharp
let left = int equationMatch.Groups.["left"].Value

let folder result (operation: Match) = 
    let right = int operation.Groups.["right"].Value
    match operation.Groups.["operator"].Value with 
    | "plus"          -> result + right
    | "minus"         -> result - right
    | "multiplied by" -> result * right
    | "divided by"    -> result / right

let result = Seq.fold folder left (Seq.cast operationMatches)
Some result
```

Note that we have to add a type annotation on the `operation` parameter and use `Seq.cast` on the `operationMatches` value, due to the `MatchCollection`
type not implementing `IEnumerable<Match>`. The result is just your typical fold and is quite readable.

Let’s run our tests to see if they now pass, which they do!

### Test 9: add then subtract

The next test also involves multiple operations, but this time using two different operations:

```fsharp
[<Test>]
let ``Can add then subtract`` () =
    Assert.That(solve "What is 1 plus 5 minus -2?", Is.EqualTo(Some 8))
```
    
Luckily for us, our fold solution already supports multiple different operations, so this test passes without any additional work.

### Test 10: subtract twice

A double subtraction is also no problem for our implementation, the test passes.

### Test 11: subtract then add

Subtracting and then adding? Already working!

### Test 12: multiply twice

Our code also immediately passes the double multiplication test.

### Test 13: add then multiply

In test 13, we first add a number and then multiply:

```fsharp
[<Test>]
let ``Can add then multiply`` () =
    Assert.That(solve "What is -3 plus 7 multiplied by -2?", Is.EqualTo(Some -8))
```
    
Although our test passes, there is something odd about this test. Shouldn’t the result be `-17`?
Well, yes and no. In mathematics, the multiplication operator has a higher precedence than the addition operator, so the equation should be evaluated as `-3 + (7 * -2)`,
which equals `-17`. However, this does not hold for this exercise, as the README explicitly states:

Remember, that these are verbal word problems, not treated as you normally would treat a written problem. This means that you calculate as you move forward each step. In other words, you should ignore order of operations. 3 + 2 * 3 = 15, not 9.
Ignoring the order of operations is precisely what we do, hence the passing test.

### Test 14: divide twice

This test passes.

### Test 15: ignore unsupported operation

With test 15, we have our first test that is expected to “fail”, indicated by returning `None`.

```fsharp
[<Test>]
let ``Cubed is too advanced`` () =
    Assert.That(solve "What is 53 cubed?", Is.EqualTo(None))
```
    
At the moment, our code does not pass this test. Why is that? Well, the left part of the equation (`"53"`) matches.
The operations regular expression does not match any operation, so it returns an empty sequence. If we fold on an empty sequence,
the initial state (`53`), is returned, hence the value returned by our code equals `Some 53`, not `None`.

The fix is simple, we just return `None` if there are no matched operations:

```fsharp
if operationMatches.Count = 0 then
    None
```
    
With this fix, the test passes.

### Test 16: irrelevant problems are not valid

The final test also deals with invalid input:

```fsharp
[<Test>]
let ``Irrelevant problems are not valid`` () =
    Assert.That(solve "Who is the president of the United States?", Is.EqualTo(None))
```
    
As there are no matching operations, `None` is returned and this test passes. With that our implementation now passes all tests. Hurray!

Does this mean we’re done? Well, we could use the CLI to submit our solution to Exercism and move on the next exercise. However, let’s try to do some refactoring/cleanup, to see if we can improve the code even more.

**Refactoring: fixing compile-time warning**

Those of your that know your F#, will have known that when we introduced the match statement, we introduced the following compile-time warning:

```
Warning FS0025: Incomplete pattern matches on this expression.
```

What this error message tells us, is that our `match` statement doesn’t handle all possible cases. This is true of course, as we only handle the four supported operators. We could ignore this warning (the code still compiles after all), but it is always a good idea to heed compiler warnings, so let’s fix it:

```fsharp
match operation.Groups.["operator"].Value with 
| "plus"          -> result + right
| "minus"         -> result - right
| "multiplied by" -> result * right
| "divided by"    -> result / right
| _               -> failwith "Unsupported operator"
```

The added default case simply throws an exception. If we now compile our code, the warning disappears.

**Refactoring: regular expression matching, revisited**

Previously, we had problems to match both the left *and* right part of an equation using a single regular expression. We then resorted to matching the equation in two parts: first the equation, then the operations. However, after some more searching, we came across a very interesting regular expressions feature: positive lookahead assertions. With that, we can actually get it all working in a single regex:

```fsharp
let solve (question: string) =
    let equationPattern = "(?<left>-?\d+) (?<operator>-?plus|minus|divided by|multiplied by) (?=(?<right>-?\d+))"
    let equationMatches = Regex.Matches(question, equationPattern)

    if equationMatches.Count = 0 then
        None
    else
        let left = int equationMatches.[0].Groups.["left"].Value

        let folder result (operation: Match) = 
            let right = int operation.Groups.["right"].Value
            match operation.Groups.["operator"].Value with 
            | "plus"          -> result + right
            | "minus"         -> result - right
            | "multiplied by" -> result * right
            | "divided by"    -> result / right
            | _               -> failwith "Unsupported operator"

        let result = Seq.fold folder left (Seq.cast equationMatches)
        Some result
```
        
**Refactoring: extracting functions**

At the moment, our `solve` function is quite big and has some repetition. Let’s remove some of the regular expression group matching code duplication by extracting two helper functions:

```fsharp
let regexGroupString (groupName: string) (m: Match) = m.Groups.[groupName].Value
let regexGroupInt    (groupName: string) (m: Match) = regexGroupString groupName m |> int

let solve (question: string) =
    let equationPattern = "(?<left>-?\d+) (?<operator>-?plus|minus|divided by|multiplied by) (?=(?<right>-?\d+))"
    let equationMatches = Regex.Matches(question, equationPattern)

    if equationMatches.Count = 0 then
        None
    else
        let left = regexGroupInt "left" equationMatches.[0]

        let folder result (operation: Match) = 
            let right = regexGroupInt "right" operation
            match regexGroupString "operator" operation with 
            | "plus"          -> result + right
            | "minus"         -> result - right
            | "multiplied by" -> result * right
            | "divided by"    -> result / right
            | _               -> failwith "Unsupported operator"

        let result = Seq.fold folder left (Seq.cast equationMatches)
        Some result
```

Better, but we still have to deal with the regular expressions groups in the `solve` function, which should be an implementation detail the `solve` function does not know about. Let’s extract the parsing of the various parts into helper functions:

```fsharp
let parseLeftOperand  (matches: MatchCollection) = regexGroupInt "left" matches.[0]
let parseRightOperand (m: Match) = regexGroupInt "right" m
let parseOperator     (m: Match) = regexGroupString "operator" m
let parseOperation    (m: Match) = (parseOperator m, parseRightOperand m)

let parseOperations (matches: MatchCollection) =
    matches 
    |> Seq.cast 
    |> Seq.map parseOperation
```
    
The relevant part of the `solve` function already looks much cleaner without all the implementation details:

```fsharp
let left = parseLeftOperand equationMatches
let operations = parseOperations equationMatches

let folder result (operator, right) = 
    match operator with 
    | "plus"          -> result + right
    | "minus"         -> result - right
    | "multiplied by" -> result * right
    | "divided by"    -> result / right
    | _               -> failwith "Unsupported operator"

let result = Seq.fold folder left operations
Some result
```

Our next step is to extract the equation result calculating code into its own function:

```fsharp
let solveEquation (left, operations) = 
    let folder result (operation, right) = 
        match operation with 
        | "plus"          -> result + right
        | "minus"         -> result - right
        | "multiplied by" -> result * right
        | "divided by"    -> result / right
        | _               -> failwith "Unsupported operator"

    Seq.fold folder left operations
```
    
The folder helper function is badly named, so let’s extract it to a better-named function:

```fsharp
let applyOperation acc (operation, right) = 
    match operation with 
    | "plus"          -> acc + right
    | "minus"         -> acc - right
    | "multiplied by" -> acc * right
    | "divided by"    -> acc / right
    | _               -> failwith "Unsupported operator"

let solveEquation (left, operations) = Seq.fold applyOperation left operations
```

At the moment, the equation solving has been extracted to a separate function, but the parsing has not. So let’s fix this:

```fsharp
let parseEquation (question: string) =
    let equationPattern = "(?<left>-?\d+) (?<operator>-?plus|minus|divided by|multiplied by) (?=(?<right>-?\d+))"
    let equationMatches = Regex.Matches(question, equationPattern)

    if equationMatches.Count = 0 then
        None
    else
        Some (parseLeftOperand equationMatches, parseOperations equationMatches)
```
        
Let’s extract the equation matching into a separate `Regex` instance:

```fsharp
let equationRegex = new Regex("(?<left>-?\d+) (?<operator>-?plus|minus|divided by|multiplied by) (?=(?<right>-?\d+))")

let parseEquation (question: string) =
    let equationMatches = equationRegex.Matches question

    if equationMatches.Count = 0 then
        None
    else
        Some (parseLeftOperand equationMatches, parseOperations equationMatches)
```
        
In general, using an `if`-statement is discouraged, so let’s use a `match` expression:

```fsharp
let parseEquation (question: string) =
    match equationRegex.Matches question with
    | matches when matches.Count = 0 -> None
    | matches -> Some (parseLeftOperand matches, parseOperations matches)
```
    
The `solve` function then becomes quite simple:

```fsharp
let solve (question: string) =
    match parseEquation question with
    | Some (left, operations) -> 
        let result = solveEquation (left, operations)
        Some result
    | None -> None
```
    
Clearly, we don’t need to (de)construct the equation tuple:

```fsharp
let solve (question: string) =
    match parseEquation question with
    | Some equation ->
        let result = solveEquation equation
        Some result
    | None -> None
```
    
Better, but this matching of the `parseEquation` result still looks a bit ugly. However, with the `Option.map` function and the pipe operator, we can rewrite this to:

```fsharp
let solve (question: string) =
    question
    |> parseEquation
    |> Option.map solveEquation
```
    
Not only does this look very pretty, it also perfectly illustrates the pipeline like nature of our implementation:

1. Parse the question into an equation.
1. Solve the equation, if it could be parsed

Through a series of relatively small refactorings, we’ve managed to create a relatively simple implementation that passes all tests. This is the complete, refactored implementation:

```fsharp
let equationRegex = new Regex("(?<left>-?\d+) (?<operator>-?plus|minus|divided by|multiplied by) (?=(?<right>-?\d+))")

let regexGroupString (groupName: string) (m: Match) = m.Groups.[groupName].Value
let regexGroupInt    (groupName: string) (m: Match) = regexGroupString groupName m |> int

let parseLeftOperand  (matches: MatchCollection) = regexGroupInt "left" matches.[0]
let parseRightOperand (m: Match) = regexGroupInt "right" m
let parseOperator     (m: Match) = regexGroupString "operator" m
let parseOperation    (m: Match) = (parseOperator m, parseRightOperand m)

let parseOperations (matches: MatchCollection) =
    matches 
    |> Seq.cast
    |> Seq.map parseOperation

let parseEquation (question: string) =
    match equationRegex.Matches question with
    | matches when matches.Count = 0 -> None
    | matches -> Some (parseLeftOperand matches, parseOperations matches)

let applyOperation acc (operation, right) = 
    match operation with 
    | "plus"          -> acc + right
    | "minus"         -> acc - right
    | "multiplied by" -> acc * right
    | "divided by"    -> acc / right
    | _               -> failwith "Unsupported operator"

let solveEquation (left, operations) = Seq.fold applyOperation left operations

let solve (question: string) =
    question
    |> parseEquation 
    |> Option.map solveEquation
```
    
While this implementation is not too bad, there is a fair amount of boilerplate code to handle the regular expression. Furthermore, we use a relatively obscure regular expression feature (positive lookahead assertions), which lessens its readability.

Is there anything we can do to fix these issues? Well, actually there is! Let’s prepare ourselves for another big rewrite, this time using parser combinators.

**Refactoring: parser combinators**

If we look at our existing implementation, we can see that we parse an equation in several, small steps:

1. Parse the left operand.
1. Parse the list of right operator/operand combinations:
  * Parse the right operator.
  * Parse the right operand.

This type of parsing setup is ideal for a specific type of text parsing library: a [parser combinator](https://en.wikipedia.org/wiki/Parser_combinator).
At its core, a parser combinator is a collection of fairly basic text-parsing functions. The reason it is called a *parser combinator*,
is that you can create complex parsers by combining those basic parsers. Let’s convert our existing, regular expression-based implementation to one that uses a parser combinator library.

Let’s start by installing the [FParsec](http://www.quanttec.com/fparsec/) parser combinator library:

```
Install-Package FParsec
```

We are now ready to create our first parser.


**Parser: left operand**

First up: parsing the left operand, which parses a `string` into an `int`. As you might have expected, FParsec already has a function to do this for you: `pint32` (which you should read as: *parse an int32*).

To verify the `pint32` parser’s behavior, open an F# interactive window and load the FParsec library references. We can then test the parser using the `run` function:

```fsharp
open FParsec

run pint32 "13";;
 val it : ParserResult<int32,unit> = Success: 13

run pint32 "789";;
 val it : ParserResult<int32,unit> = Success: 789

run pint32 "abc";;
 val it : ParserResult<int32,unit> =
   Failure:
 Error in Ln: 1 Col: 1
 abc
 ^
 Expecting: integer number (32-bit, signed)
```
 
The first two inputs were valid `int32` values, so the run function returned a `Success` value containing the corresponding, parsed number.
The third input was not a correct `int32` value, so a `Failure` value was returned containing an error message.

Defining our left operand parser is simple:

```fsharp
let parseLeftOperand = pint32
```

**Parser: right operand**

Similarly, we can define our right operand parser as:

```fsharp
let parseRightOperand = pint32
```

Hmmm, the `parseLeftOperand` and `parseRightOperand` functions are identical! We can safely merge them into a single function:

```fsharp
let parseOperand = pint32
```

Previously, this was not possible, as the operand values had to be retrieved from differently named regular expression groups.

**Parser: right operator**

Our right operator parser is one that matches one of the following string values:

* "plus"
* "minus"
* "divided by"
* "multiplied by"

Any other string value is considered invalid. We can create a parser to match a specific string using the `pstring` function:

```fsharp
run (pstring "abc") "abc";;
 val it : ParserResult<string,unit> = Success: "abc"

run (pstring "abc") "abcde";;
 val it : ParserResult<string,unit> = Success: "abc"

run (pstring "abc") "cde";;
 val it : ParserResult<string,unit> =
   Failure:
 Error in Ln: 1 Col: 1
 cde
 ^
 Expecting: 'abc'
```
 
Okay, so we can create a parser to match a string, but how to create a parser that matches one of the four operators? Well, simple: we use the `<!>` (read: or) parser:

```fsharp
let parseOperator = 
    pstring "plus"          <|> 
    pstring "minus"         <|>
    pstring "divided by"    <|>
    pstring "multiplied by"
```
    
Here, we use the `<|>` operator to specify that should its left hand parser fail to parse, it should try to parse its right hand parser.
Let’s send this function to the F# interactive to be able to test it:

```
  Stopped due to error
 System.Exception: Operation could not be completed due to earlier error
 Value restriction. The value 'parseOperator' has been inferred to have generic type
     val parseOperator : Parser<string,obj>    
 Either make the arguments to 'parseOperator' explicit or, if you do not intend for it to be generic, add a type annotation.
```
 
Ouch, we get a very weird-looking error! What’s going on? Well, don’t worry, this is a [common error](http://www.quanttec.com/fparsec/tutorial.html#fs-value-restriction) when using FParsec.
The problem is that the `parseOperator` function is of type `Parser<string,'_a>`, but F# doesn’t allow allow a generic type parameter here (`'_a` must be a concrete type). We can do two things to fix this:

1. Add a type annotation.
1. Help the compiler infer the type.

We’ll go for the second option, by also sending a run call to the F# interactive window:

```fsharp
let parseOperator = 
    pstring "plus"          <|> 
    pstring "minus"         <|>
    pstring "divided by"    <|>
    pstring "multiplied by"

run parseOperator "plus"
 val it : ParserResult<string,unit> = Success: "plus"

run parseOperator "divided by";;
 val it : ParserResult<string,unit> = Success: "divided by"

run parseOperator "div";;
 val it : ParserResult<string,unit> =
   Failure:
 Error in Ln: 1 Col: 1
 div
 ^
 Expecting: 'divided by', 'minus', 'multiplied by' or 'plus'
```
 
As can be seen, the parser works as expected: it matches if the supplied input equals one of the four possible string values.

**Parser: right operator/operand combination**

The parser for the right operator/operand combination has to do the following:

1. Match the operator.
1. Match a single space.
1. Match the operand.
1. Return the operator/operand combination.

Let’s start with step 1, matching the operator:

```fsharp
let parseOperation =
    parseOperator
```
    
The next step is to match on a single space, which we can do using `pchar ' '`.
So how do we combine these two parsers? We cannot use the `<|>` operator, as that is an **or** match, not an **and**.
As it turns out, we can use several operators. To choose between these operators, we have to decide which parser results we want to retain.
Obviously, the matched operator is important, but the single space is not. Therefore, we should use the parser operator that returns the first parser
and ignores the second parser. For this purpose, we can use the `.>>` operator:

```fsharp
let parseOperation =
    parseOperator .>> pchar ' '
```
    
The third and fourth step can be combined into a single call to the `.>>.` operator, which returns the results of both parsers:

```fsharp
let parseOperation =
    parseOperator .>> pchar ' ' .>>. parseOperand
```
    
Let’s test this parser:

```fsharp
run parseOperation "plus 7";;
 val it : ParserResult<(string * int32),unit> = Success: ("plus", 7)

run parseOperation "minus 345";;
 val it : ParserResult<(string * int32),unit> = Success: ("minus", 345)

run parseOperation "minus plus";;
 val it : ParserResult<(string * int32),unit> =
   Failure:
 Error in Ln: 1 Col: 7
 minus plus
       ^
 Expecting: integer number (32-bit, signed)
```
 
You can see that the result of the parser is a tuple, with the first item being the result of the operator parser, and the second item the result of the operand parser.

**Parser: list of right operator/operand combinations**

Now that we have a parser for a right operator/operand combination, the next step is to create a parser that can match one or more of those combinations.
Once again, there is an existing parser that can do that: the [many1 parser](http://www.quanttec.com/fparsec/reference/primitives.html#members.many1).
This parser function takes another parser, and matches one of more instances of the supplied parser argument.

Our initial attempt looked like this:

```fsharp
let parseOperations = many1 parseOperation
```

Let’s test this parser:

```fsharp
run parseOperations "plus 3 minus 4";;
 val it : ParserResult<(string * int32) list,unit> = Success: [("plus", 3)]

run parseOperations "plus 7 multiplied by -2";;
 val it : ParserResult<(string * int32) list,unit> = Success: [("plus", 7)]
```
 
Hmmm, the parser does match successfully, but only the first combination. Why is that? Well, there is a very simple explanation for this.
When parsing the `"plus 3 minus 4"` input, it finds a match (`"plus 3"`). It then tries to parse the remaining input: `" minus 4"`.
Did you notice the leading space? That’s what prevents the second combination from being matched, as the `parseOperations` parser expects the input
to start with the operator, not a space.

To fix this, we can do two things:

1. Modify the `parseOperation` to ignore leading and/or trailing spaces.
1. Use the `sepBy1` parser, which matches a parser one or more times using another parser as a separator.

We’ll go with the second option:

```fsharp
let parseOperations = sepBy1 parseOperation (pchar ' ')

run parseOperations "plus 3 minus 4";;
 val it : ParserResult<(string * int32) list,unit> =
   Success: [("plus", 3); ("minus", 4)]

run parseOperations "plus 7 multiplied by -2";;
 val it : ParserResult<(string * int32) list,unit> =
   Success: [("plus", 7); ("multiplied by", -2)]

run parseOperations "multiplied by 6";;
 val it : ParserResult<(string * int32) list,unit> =
   Success: [("multiplied by", 6)]
```
   
The modified `parseOperations` parser now correctly parses a list of operator/operand tuples.

**Parser: equation**

The final step is to create the equation parser, which combines the left and right parsers and ignores the pre- and postfix strings:

```fsharp
let parseEquation = 
    pstring "What is "
     >>. parseOperand
    .>>  pchar ' '
    .>>. parseOperations
    .>>  pstring "?"
```

Let’s test this parser:

```fsharp
run parseEquation "What is -12 divided by 2 divided by -3?";;
 val it : ParserResult<(int32 * (string * int32) list),unit> =
   Success: (-12, [("divided by", 2); ("divided by", -3)])

run parseEquation "What is 17 minus 6 plus 3?";;
 val it : ParserResult<(int32 * (string * int32) list),unit> =
   Success: (17, [("minus", 6); ("plus", 3)]) 
```
   
**Integrating the parser**

We can now use our FParsec-based equation parser in the `solve` function:

```fsharp
let parseToOption parser (input: string) =
    match run parser input with
    | Success(result, _, _)   -> Some result
    | Failure(errorMsg, _, _) -> None

let solve (question: string) =
    parseToOption parseEquation question
    |> Option.map solveEquation
```
    
Let’s try to run our tests, and hurray, all tests pass! For reference, this is the full FParsec-based implementation:

```fsharp
open FParsec

let parseOperand = pint32

let parseOperator = 
    pstring "plus"          <|> 
    pstring "minus"         <|>
    pstring "divided by"    <|>
    pstring "multiplied by"

let parseOperation =
    parseOperator .>> pchar ' ' .>>. parseOperand

let parseOperations = sepBy1 parseOperation (pchar ' ')

let parseEquation = 
    pstring "What is "
     >>. parseOperand
    .>>  pchar ' '
    .>>. parseOperations
    .>>  pstring "?"

let parseToOption parser (input: string) =
    match run parser input with
    | Success(result, _, _)   -> Some result
    | Failure(errorMsg, _, _) -> None

let applyOperation acc (operation, right) = 
    match operation with 
    | "plus"          -> acc + right
    | "minus"         -> acc - right
    | "multiplied by" -> acc * right
    | "divided by"    -> acc / right
    | _               -> acc

let solveEquation (left, operations) = Seq.fold applyOperation left operations

let solve (question: string) =
    parseToOption parseEquation question
    |> Option.map solveEquation
```
    
The only slightly cryptic part of this implementation are the operators: `<|>`, `>>.`, `.>>` and `.>>.`.
Someone new to FParsec will probably not intuitively understand what they do, but to me that’s the only real hard part of this code. The rest is actually quite easy to read (and write).

**Refactoring: using an Abstract Syntax Tree**

At the moment, we parse an equation into very simple types: strings and integers. Ideally, one should define an [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) (AST)
to better describe the structure of our equation. As it turns out, this is a piece of cake in F#, so let’s have a go!

First, we define types for each type of syntax node in our equation. This is a perfect use case for [discriminated unions](https://fsharpforfunandprofit.com/posts/discriminated-unions/). Let’s start with an operand:

```fsharp
type Operand = Int of int
```

Simple, an operand is a discriminated union with only a single case: `Int`, which contains an `int` value. Defining an operand this way allows us to easily add other types of operands later, like floating point numbers, booleans or variables.

Next, we define the type to represent the four supported operators:

```fsharp
type Operator = 
    | Plus
    | Minus
    | DividedBy
    | MultipliedBy 
```
    
We then define an operation as an `Operator`/`Operand` tuple:

```fsharp
type Operation = Operation of Operator * Operand
```

Finally, we define the equation:

```fsharp
type Equation = Equation of Operand * Operation list
```

Note that we haven’t specified types for the leading and trailing strings, nor for the spaces, as they are not relevant to our problem.

The following step is to use these types in our implementation.

**Use AST types in parseOperand**

Let’s start with the `parseOperand` function, which should wrap the parsed `int` value in an `Operand` type. We can do this using the `|>>` (read: pipe) operator:

```fsharp
let parseOperand = pint32 |>> Int

run parseOperand "54";;
 val it : ParserResult<Operand,unit> = Success: Operand 54

run parseOperand "123";;
 val it : ParserResult<Operand,unit> = Success: Operand 123

run parseOperand "ab";;
val it : ParserResult<Operand,unit> =
   Failure:
 Error in Ln: 1 Col: 1
 ab
 ^
 Expecting: integer number (32-bit, signed)
```
 
Perfect.

**Use AST types in parseOperator**

Next up: the `parseOperator` parser:

```fsharp
let parseOperator = 
    (pstring "plus"         |>> (fun _ -> Plus))         <|>
    (pstring "minus"        |>> (fun _ -> Minus))        <|>
    (pstring "divided by"   |>> (fun _ -> DividedBy))    <|>
    (pstring "multiplied by"|>> (fun _ -> MultipliedBy))

run parseOperator "plus"
 val it : ParserResult<Operator,unit> = Success: Plus

run parseOperator "divided by"
 val it : ParserResult<Operator,unit> = Success: DividedBy

run parseOperator "power"
 val it : ParserResult<Operator,unit> =
   Failure:
 Error in Ln: 1 Col: 1
 power
 ^
 Expecting: 'divided by', 'minus', 'multiplied by' or 'plus'
```
 
Everything works, but the pipe operator usage looks a bit odd. This is due to the pipe operator expecting a `string -> 'a` function as its argument.
However, we’re not interested in the matched `string` value, so we ignore it and return the appropriate operator.
As it turns out, the `>>%` operator simplifies this by not expecting a `string` parameter:

```fsharp
let parseOperator = 
    (pstring "plus"          >>% Plus)         <|>
    (pstring "minus"         >>% Minus)        <|>
    (pstring "divided by"    >>% DividedBy)    <|>
    (pstring "multiplied by" >>% MultipliedBy)
```
    
Much better, but there is an even better parser: `stringReturn`:

```fsharp
let parseOperator = 
    stringReturn "plus"          Plus         <|>
    stringReturn "minus"         Minus        <|>
    stringReturn "divided by"    DividedBy    <|>
    stringReturn "multiplied by" MultipliedBy
```
    
Excellent! Clean, simple and fully functional!

**Use AST types in parseOperation**

With what we know now, modifying the `parseOperation` to return an `Operation` type is simple:

```fsharp
let parseOperation =
    parseOperator .>> pchar ' ' .>>. parseOperand |>> Operation

run parseOperation "plus 1"
 val it : ParserResult<Operation,unit> = Success: Operation (Plus,Operand 1)
```
 
**Use AST types in parseOperations**

The `parseOperations` function already returns the correct type (`Operation list`), so we don’t need to modify it:

```fsharp
run parseOperations "plus 4 minus 6"
 val it : ParserResult<Operation list,unit> =
   Success: [Operation (Plus,Operand 4); Operation (Minus,Operand 6)]
```
   
**Use AST types in parseEquation**

For the `parseEquation`, we can just pipe the result into the `Equation` type:

```fsharp
let parseEquation = 
    pstring "What is "
     >>. parseOperand
    .>>  pchar ' '
    .>>. parseOperations
    .>>  pstring "?"
    |>> Equation

run parseEquation "What is -12 divided by 2 divided by -3?"
 val it : ParserResult<Equation,unit> =
   Success: Equation
   (Operand -12,
    [Operation (DividedBy,Operand 2); Operation (DividedBy,Operand -3)])
```
    
As we now use the `pchar ' '` parser in several places, we should really define a separate parser for it:

```fsharp
let space = pchar ' '
```

**Use AST types when solving the solution**

The final step is to use the AST types when we solve the equation:

```fsharp
let applyOperation acc (Operation (operator, Operand(right))) = 
    match operator with 
    | Plus         -> acc + right
    | Minus        -> acc - right
    | MultipliedBy -> acc * right
    | DividedBy    -> acc / right
    | _            -> acc

let solveEquation (Equation (Operand(left), operations)) = Seq.fold applyOperation left operations
```

To simplify our code, we pattern-match directly in our parameters. At this point, we run our tests and they all pass! Hurray.

As we now use a discriminated union to represent an operator, the compiler is able to inform us that the catch-all clause will never be reached. We can thus safely remove it.
The complete code then becomes:

```fsharp
open FParsec

type Operand = Operand of int

type Operator = 
    | Plus
    | Minus
    | DividedBy
    | MultipliedBy

type Operation = Operation of Operator * Operand

type Equation = Equation of Operand * Operation list

let space = pchar ' '

let parseOperand = pint32 |>> Operand

let parseOperator = 
    stringReturn "plus"          Plus         <|>
    stringReturn "minus"         Minus        <|>
    stringReturn "divided by"    DividedBy    <|>
    stringReturn "multiplied by" MultipliedBy

let parseOperation =
    parseOperator .>> space .>>. parseOperand |>> Operation

let parseOperations = sepBy1 parseOperation space

let parseEquation = 
    pstring "What is "
     >>. parseOperand
    .>>  space
    .>>. parseOperations
    .>>  pstring "?"
    |>> Equation

let parseToOption parser (input: string) =
    match run parser input with
    | Success(result, _, _)   -> Some result
    | Failure(errorMsg, _, _) -> None

let applyOperation acc (Operation (operator, Operand(right))) = 
    match operator with 
    | Plus         -> acc + right
    | Minus        -> acc - right
    | MultipliedBy -> acc * right
    | DividedBy    -> acc / right

let solveEquation (Equation (Operand(left), operations)) = Seq.fold applyOperation left operations

let solve (question: string) =
    parseToOption parseEquation question
    |> Option.map solveEquation
```
    
To me, what’s remarkable is that we defined and implemented an AST for our equation with only very code and hardly any effort.

**Refactoring: using a different AST**

Our previous AST didn’t look too bad, but it might not be the best way to describe an equation.
Consider the expression `17 minus 6 plus 3`. At the moment, our corresponding AST looks like this:

```fsharp
Equation(
    Operand 17 * 
        [ Operation(Minus, Operand 6); 
          Operation(Plus, Operand 3) ])
```
          
What this AST fails to represent, is that our operators always operate on two operands: a left and right part.
What we would like is `17 minus 6` to be defined as `Minus (Operand 17, Operand 6)`.
The multiple operations in `17 minus 6 plus 3` should be defined as: `Plus (Minus (Operand 17, Operand 6), Operand 3)`.
Note that this is an inside-out representation of an equation, the left-most part of the equation is the innermost part of the AST.
Solving this AST is thus a recursive, inside-out algorithm:


1. `Plus (Minus (Operand 17, Operand 6), Operand 3)`,
1. `Plus (Operand 11, Operand 3)`,
1. `Operand 14`


As it turns out, defining this AST in our code is fantastically simple:

```fsharp
type Expression = 
    | Operand of int
    | Plus of Expression * Expression
    | Minus of Expression * Expression
    | DividedBy of Expression * Expression
    | MultipliedBy of Expression * Expression
```
    
**Use updated AST in parseOperand**

The `parseOperand` function does not need to be changed. The only difference is that its returned value (`Operand`) is now part of the `Expression` discriminated union.

**Use updated AST in parseOperator**

There is also no need to modify the `parseOperator` function.

**Use updated AST in parseOperation and parseOperations**

The `parseOperation` is where things start getting interesting. Previously, an operation was just an operator and an operand, but the operand can now also be another operation. Our initial attempt might look like this:

```fsharp
let parseOperation = 
    parseExpression 
    .>>  space
    .>>. sepBy1 (parseOperator .>> space .>>. parseExpression) space
    |>> TODO

let parseExpression = parseOperand <|> parseOperation
```

In this modified `parseOperation` function, we use the new `parseExpression` parser to parse the left and right hand sides.
This new parser then matches on either an operand, or an operation. Unfortunately, we have now introduced a cycle: `parseOperation` must know of `parseExpression`,
but also the other way around. How can we solve this? Enter the `createParserForwardedToRef` function.

When you call `createParserForwardedToRef`, it returns two things:

* A parser `p` which type will be inferred depending on its usage.
* A parser reference cell `pRef`, to which parser `p` forwards all calls.

With this function, we can start using the parser `p`, but defer binding its actual implementation until after all required parsers have been defined.
Our modified code looks like this:

```fsharp
let expr, exprImpl = createParserForwardedToRef()

let parseOperation = 
    expr 
    .>>  space 
    .>>. sepBy1 (parseOperator .>> space .>>. expr) space
    |>> TODO

let parseExpression = parseOperand <|> parseOperation

exprImpl := parseExpression   
```

You can see that we use the `createParserForwardedToRef` to create a parser and its reference cell. We then use this parser in our `parseOperation` function,
and later bind the `parseExpression` function to the reference cell’s contents. The end result is precisely what we wanted to achieve.

We now have to decide how to process the parsed operation using the `|>>` operator.
If you are starting to become more proficient at reading FParsec’s operators, you will have guessed that we have to process a tuple. The first item of this tuple is the left expression, and the second item is the list of operator/right expression combinations. The first step is thus:

```fsharp
|>> (fun (left, operations) -> TODO)
```

So how can we merge these two parts into a single expression? Well, we can use a regular fold:

```fsharp
List.fold (fun acc (operator, right) -> operator (acc, right)) left operations
```

The initial state is the left expression. We then process each operator in sequence, wrapping the previous expression in the new operator.
The end result will be an `Expression` that is structured inside-out, precisely what we want.

It’s probably best to extract the expression building code into its own function. This gives us the following code:

```fsharp
let buildExpression (left, operations) =
    List.fold (fun acc (operator, right) -> operator (acc, right)) left operations

let parseOperation = 
    expr 
    .>>  space 
    .>>. sepBy1 (parseOperator .>> space .>>. expr) space
    |>> buildExpression
```
    
**Use updated AST in parseEquation**

The `parseEquation` function is very simple, it just uses the parseOperation parser to parse the equation:

```fsharp
let parseEquation = pstring "What is " >>. parseOperation .>> pstring "?"
```

**Use updated AST in solveExpression**

The benefit of our modified AST becomes apparent when we work on our equation solving code. As an operator in our updated AST now knows about its operands, solving an expression is a simple, recursive function:

```fsharp
let rec solveExpression =
    function
    | Operand x -> x
    | Plus (left, right)         -> solveExpression left + solveExpression right
    | Minus (left, right)        -> solveExpression left - solveExpression right
    | DividedBy (left, right)    -> solveExpression left / solveExpression right
    | MultipliedBy (left, right) -> solveExpression left * solveExpression right
```
    
I think this function is very readable and elegant.

**Use updated AST in solve**

The `solve` function does not need any modification, so we can now run all tests, which all pass!

This is the complete, updated implementation:

```fsharp
open FParsec

type Expression = 
    | Operand of int
    | Plus of Expression * Expression
    | Minus of Expression * Expression
    | DividedBy of Expression * Expression
    | MultipliedBy of Expression * Expression

let buildExpression (left, operations) =
    List.fold (fun acc (operator, right) -> operator (acc, right)) left operations

// We create a parser forwarder in order to allow us to 
// define a recursive parser later on
let expr, exprImpl = createParserForwardedToRef()

let space = pchar ' '

let parseOperand = pint32 |>> Operand

let parseOperator = 
    stringReturn "plus"          Plus         <|> 
    stringReturn "minus"         Minus        <|> 
    stringReturn "divided by"    DividedBy    <|> 
    stringReturn "multiplied by" MultipliedBy

let parseOperation = 
    expr 
    .>> space 
    .>>. sepBy1 (parseOperator .>> space .>>. expr) space
    |>> (fun (left, operations) -> 
        List.fold (fun acc (operator, right) -> operator (acc, right)) left operations)

let parseExpression = parseOperand <|> parseOperation

exprImpl := parseExpression

let parseEquation =  pstring "What is " >>. parseOperation .>> pstring "?"

let parseToOption parser (input: string) =
    match run parser input with
    | Success(result, _, _)   -> Some result
    | Failure(errorMsg, _, _) -> None

let rec solveExpression =
    function
    | Operand x -> x
    | Plus (left, right) -> solveExpression left + solveExpression right
    | Minus (left, right) -> solveExpression left - solveExpression right
    | DividedBy (left, right) -> solveExpression left / solveExpression right
    | MultipliedBy (left, right) -> solveExpression left * solveExpression right

let solve (question: string) =
    parseToOption parseEquation question
    |> Option.map solveExpression
```
    
I think that this AST implementation is more elegant than our previous one, and also leads to more readable code.

## Bonus: implementations in Scala and Haskell

One of the great things about Exercism is that almost all exercises are available in more than one language. This also goes for the wordy exercise. We will list our Scala and Haskell implementations of the wordy exercise to allow you to compare them to our F# implementation.

### Bonus: Scala implementation

First up: our Scala implementation of the wordy problem:

```scala
import scala.util.parsing.combinator._

sealed trait Expression {
  def solve: Int
}
case class Operand(n: Int) extends Expression {
  def solve = n
}
case class Sum(e1: Expression, e2: Expression) extends Expression {
  def solve = e1.solve + e2.solve
}
case class Subtract(e1: Expression, e2: Expression) extends Expression {
  def solve = e1.solve - e2.solve
}
case class Multiply(e1: Expression, e2: Expression) extends Expression {
  def solve = e1.solve * e2.solve
}
case class Divide(e1: Expression, e2: Expression) extends Expression {
  def solve = e1.solve / e2.solve
}

object WordProblem extends RegexParsers {
  def operand: Parser[Operand] = """(0|-?[1-9]\d*)""".r ^^ { str => Operand(str.toInt) }

  def operator: Parser[String] = "plus" | "minus" | "multiplied by" | "divided by"

  def operation: Parser[Expression] =
    expression ~ (operator ~ expression).* ^^ {
      case expr ~ list => list.foldLeft(expr) {
        case (left, "plus" ~ right)          => Sum(left, right)
        case (left, "minus" ~ right)         => Subtract(left, right)
        case (left, "multiplied by" ~ right) => Multiply(left, right)
        case (left, "divided by" ~ right)    => Divide(left, right)
      }
    }

  def expression: Parser[Expression] = operand | operation

  def equation: Parser[Expression] = "What is " ~> operation <~ "?"

  def apply(input: String): Option[Int] = {
    parse(equation, input) match {
      case Success(expression, _) => Some(expression.solve)
      case _ => None
    }
  }
}
```

This implementation is actually quite similar to our F#, which is in large part due to the fact that we use Scala’s parser combinator library. There are two main differences:

* The case classes are Scala’s implementation of discriminated unions. As such, the solution is more object-oriented.
* The parser combinator library’s operators are different (`^^` instead of `|>>`).

### Bonus: Haskell implementation

For Haskell, we use the `Attoparsec` parser combinator library:

```haskell
{-# LANGUAGE OverloadedStrings #-}

module WordProblem (answer) where

import Control.Applicative ((<|>), (<$>))
import Data.Attoparsec.Text (Parser, parse, maybeResult, space, decimal, signed, string, sepBy1)
import Data.Text (pack)

data Expression 
    = Operand Int 
    | Plus Expression Expression 
    | Minus Expression Expression 
    | DividedBy Expression Expression 
    | MultipliedBy Expression Expression
    deriving (Show)

buildExpression :: Expression -> [(Expression -> Expression -> Expression, Expression)] -> Expression 
buildExpression = foldl (\acc (operator, right) -> operator acc right)

operandParser :: Parser Expression 
operandParser = Operand <$> signed decimal

operatorParser :: Parser (Expression -> Expression -> Expression)
operatorParser = 
    (string "plus"          >> return Plus)         <|>
    (string "minus"         >> return Minus)        <|>
    (string "divided by"    >> return DividedBy)    <|>
    (string "multiplied by" >> return MultipliedBy)

operationParser :: Parser Expression
operationParser = do
    left       <- (operandParser <|> operationParser) <* space
    operations <- sepBy1 (do
        operator  <- operatorParser <* space 
        operation <- operandParser <|> operationParser
        return (operator, operation)) space
    return (buildExpression left operations)

equationParser :: Parser Expression
equationParser = string "What is " *> operationParser <* string "?"

solveExpression :: Expression -> Int
solveExpression (Operand x) = x
solveExpression (Plus left right) = solveExpression left + solveExpression right
solveExpression (Minus left right) = solveExpression left - solveExpression right
solveExpression (DividedBy left right) = solveExpression left `div` solveExpression right
solveExpression (MultipliedBy left right) = solveExpression left * solveExpression right

answer :: String -> Maybe Int
answer = fmap solveExpression . maybeResult . parse equationParser . pack
```

Once again, there are some syntactic differences and some different operators, but the gist of it remains the same.
The only significant difference is that we use [do notation](http://learnyouahaskell.com/a-fistful-of-monads#do-notation),
a feature similar to Scala’s [for comprehensions](http://docs.scala-lang.org/tutorials/FAQ/yield) and F#’s [computation expressions](http://fsharpforfunandprofit.com/posts/computation-expressions-intro/).

### Bonus: operator precedence

As a final bonus, we’ll look at what we would have to do if the wordy problem did respect the [standard operator precedence rules](https://en.wikipedia.org/wiki/Order_of_operations).
You might remember that we talked about this when we discussed test 13, which required us to solve `"What is -3 plus 7 multiplied by -2?"`.
Due to the exercise ignoring the operator precedence, the result was `(-3 + 7) * -2 = -8`. However, if we would use the standard convention
that multiplication has a higher precedence than addition, the result should be `-3 + (7 * -2) = -17`.
Let’s modify our implementation to follow the standard operator precedence rules.

Luckily for us, we don’t have to write any operator precedence handling code, FParsec has built-in capabilities for operator precedence handling.
The basis is formed by the `OperatorPrecedenceParser` class. The first step is thus to create an `OperatorPrecedenceParser` instance,
supplying it with the type we want it to return (`int`):

```fsharp
let opp = new OperatorPrecedenceParser<int,unit,unit>()
```

Note: you can ignore the second and third type parameter, we won’t be needing them.

The next step is to define how terms (which we referred to as operand) should be parsed:

```fsharp
opp.TermParser <- pint32 .>> spaces
```

For convenience purposes, we’ve added the `spaces` parser to our parser, which allows us to just ignore any trailing spaces.

We can now specify the operators. There are four types of operators:

* [InfixOperator](http://www.quanttec.com/fparsec/reference/operatorprecedenceparser.html#members.InfixOperator)
* [PrefixOperator](http://www.quanttec.com/fparsec/reference/operatorprecedenceparser.html#members.PrefixOperator)
* [PostfixOperator](http://www.quanttec.com/fparsec/reference/operatorprecedenceparser.html#members.PostfixOperator)
* [TernaryOperator](http://www.quanttec.com/fparsec/reference/operatorprecedenceparser.html#members.TernaryOperator)

Our equation only has infix operators, so we will be using the `InfixOperator` type. In order to create an `InfixOperator`, we have to specify five parameters:

* The operator string.
* The after-operator parser (which we’ll use to skip trailing spaces).
* The operator precedence.
* The associativity: none, left or right.
* The binary function used to combine terms.

This is how we add the operators to represent our equation:

```fsharp
opp.AddOperator(InfixOperator("plus",          spaces, 1, Associativity.Left, fun x y -> x + y))
opp.AddOperator(InfixOperator("minus",         spaces, 1, Associativity.Left, fun x y -> x - y))
opp.AddOperator(InfixOperator("multiplied by", spaces, 2, Associativity.Left, fun x y -> x * y))
opp.AddOperator(InfixOperator("divided by",    spaces, 2, Associativity.Left, fun x y -> x / y))
```

Very simple. You can see that the `"multiplied by"` and `"divided by"` operators have a higher precedence (`2`) than the `"plus"` and `"minus"` operators (`1`).

We can of course simplify the term combination function by just passing the operator:

```fsharp
opp.AddOperator(InfixOperator("plus",          spaces, 1, Associativity.Left, (+)))
opp.AddOperator(InfixOperator("minus",         spaces, 1, Associativity.Left, (-)))
opp.AddOperator(InfixOperator("multiplied by", spaces, 2, Associativity.Left, (*)))
opp.AddOperator(InfixOperator("divided by",    spaces, 2, Associativity.Left, (/)))
```

At this point, the `OperatorPrecedenceParser` instance has been correctly configured. We can then use its `ExpressionParser` property in our `parseEquation` function:

```fsharp
let parseEquation = pstring "What is " >>. opp.ExpressionParser .>>  pstring "?"
```

If we now try to solve `"What is -3 plus 7 multiplied by -2?"`, the answer will be `-17`, exactly what we expect.

The updated source that uses the `OperatorPrecedenceParser` is listed below:

```fsharp
open FParsec

let opp = new OperatorPrecedenceParser<int,unit,unit>()

opp.TermParser <- pint32 .>> spaces

opp.AddOperator(InfixOperator("plus",          spaces, 1, Associativity.Left, (+)))
opp.AddOperator(InfixOperator("minus",         spaces, 1, Associativity.Left, (-)))
opp.AddOperator(InfixOperator("multiplied by", spaces, 2, Associativity.Left, (*)))
opp.AddOperator(InfixOperator("divided by",    spaces, 2, Associativity.Left, (/)))

let parseEquation = pstring "What is " >>. opp.ExpressionParser .>>  pstring "?"

let parseToOption parser (input: string) =
    match run parser input with
    | Success(result, _, _)   -> Some result
    | Failure(errorMsg, _, _) -> None

let solve (question: string) = parseToOption parseEquation question
```

If we change the expected result for "What is -3 plus 7 multiplied by -2?" to Some -17 and run the tests again, we’ll see that all tests pass! Isn’t that amazing for a grand total of 13 lines of code?

The fun thing is, if we set the operator precedence of all operators to the same value, we have a fully valid implementation of the wordy problem!

## Building a parser combinator from scratch

As we are really starting to appreciate parser combinators, let’s build one from scratch!

Got you! You didn’t really think we were going to do that, did you? If you are interested in how one would build a parser combinator library though,
I highly recommend the [Understanding Parser Combinators](http://fsharpforfunandprofit.com/posts/understanding-parser-combinators/) series on the fantastic [F# for fun and profit](http://fsharpforfunandprofit.com/)
website. In that series, Scott Wlaschin gradually builds a fully functional parser combinator library from scratch.

## Conclusion

Solving the wordy problem was a very interesting exercise. The first few tests could be solved using just string replacing, splitting and converting. That approach quickly showed its defects, so we tried something else.

The second approach used regular expressions to parse the text. Using more advanced regular expression features (such as positive lookahead assertions), we could solve the problem using relatively little code. The readability was not great though, so it was time for another rewrite.

Our final approach used the FParsec parser combinator library. This resulted in a very elegant and concise solution. For fun, we then defined an Abstract Syntax Tree for the syntax being parsed, which was surprisingly easy and required very little code. We then showed how easy it is to define an expression parser with operator precedence rules in FParsec, further simplifying our implementation.

Of the three approaches we tried, the FParsec one was our favorite by far. It resulted in a simple, elegant implementation that was very readable. This was for a large part due to the many useful parsers FParsec includes out of the box. With these parser building blocks, we quickly and easily built our own, more complex parser. All in all, I can wholeheartedly recommend FParsec, whenever you have a text-parsing problem.
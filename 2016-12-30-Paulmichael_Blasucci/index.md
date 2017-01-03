



# Tips & Tricks to Improve Your F# Library’s Public API #

*All text and code copyright (c) 2016 by Paulmichael Blasucci. Used with permission.*

*Original post dated 2016-12-30 available at https://pblasucci.wordpress.com/2016/12/30/betterapi/*

**By Paulmichael Blasucci**


With the addition of F# in 2010, the .NET run-time gained a terrific, production-grade, functional-first programming language. This post is aimed at anyone who uses that terrific language to develop .NET libraries.

Because, you see, there’s a bit of a problem we need to discuss.

One of the popularized strengths of F# is its ability to both consume and be consumed by other .NET languages. And yet, if not done carefully, consuming F# code from, for instance, Visual Basic can be an absolute nightmare. Fortunately, over the years, I’ve settled upon a few different techniques that can mitigate, and in some cases even obliterate, any unpleasantness. Of course, this article assumes you, as a library author, *actually want* other developers to have a pleasant experience consuming your work (regardless of the .NET language they employ). If that’s not how you feel, no worries — but you can stop reading now. The rest of this post really won’t appeal to you.

What follows are 12 guidelines, listed in descending order of priority, which each address one, or more, *potentially awkward* points of integration between F# and other .NET languages. It’s important to note: this advice is intended for the *public* API of your libraries. By all means, do whatever awesome, clever things you’d like *internally*. This list is just to help give it a pretty face.

## 1. Do limit the use of advanced generic constraints

F# supports a wider range of possible generic type constraints than either C# or VB. Not matter how useful, or cool, a constraint might seem (sorry, SRTP fans), it’s meaningless if a consumer can’t possibly comply with it. To that end, public APIs should only leverage the following types of constraints:

| Constraint | Syntax |
| ---- | ---- |
| Subtype | `type-parameter :> type` |
| Constructor | `type-parameter : ( new : unit -> 'a )`  |
| Value Type | `type-parameter : struct` |
| Reference Type | `type-parameter : not struct` |

## 2. Do expose TPL rather than Async workflows

Asynchronous programming in F#, via `Async` workflows, is a simply unintelligible mess in other languages. Fortunately, it’s quite easy to integrate `System.Threading.Tasks.Task` into a workflow. Tasks received as input can be sequenced via `Async.AwaitTask`:

```fsharp
let runAll tasks = 
  let rec loop agg work = async {
    match work with
    | [  ] -> return List.rev agg
    | h::t -> let! v = Async.AwaitTask h
              let  r = v * 2
              return! loop (r::agg) t } 
  tasks |> loop []
```
  
Meanwhile, if you need to return an async workflow to a non-F# caller, you can leverage `Async.StartAsTask`:

```fsharp
let getUserPrefs conn uid =  
  async { use db = new DbUsers() 
          let! prefs = db.ExecuteAsync(conn,uid) 
          return marshal prefs }
  |> Async.StartAsTask
```
  
## 3. Prefer BCL collection types

F# ships with a small number of persistent functional collections. These are the bread and butter of functional programming. But they’re cumbersome and confusing in other languages. So, for input parameters and return values, consider converting to or from common collection types. For example, when working with `List` or `Set`:

```fsharp
(* NOT RECOMMENDED *)
let transform (value :int list) =
  // do stuff with values...
 
(* DO THIS INSTEAD *)
let transform (values :int seq) =
  let values' = Seq.toList values
  // do stuff with values' ...
```
  
Similarly, when working with `Map`:

```fsharp
(* NOT RECOMMENDED *)
let transform (table :Map<string,int>) =
  // do stuff with table ...
 
(* DO THIS INSTEAD *)
let transform (table :IDictionary<string,int>) =
  let table' = 
    table 
    |> Seq.map (function KeyValue (k,v) -> (k,v)) 
    |> Map
  // do stuff with table' ...
```
  
Note, in the previous samples, type signatures are for demonstration purposes only. Also, note that similar conversions (e.g. `Set.toSeq`, et cetera) can, *and should*, be used for return values.

## 4. Do provide conversions from standard delegates to F# functions

Owning to a number of very good technical and historical reasons, F# uses a different first-class function mechanism than other languages. And, while the F# compiler makes it pretty easy to turn (`fun x -> x * 2`) into a `Func`, the inverse is not so easy. So, it becomes important to provide some means of supporting the standard BCL delegates `Func` and `Action` (which is what C# and VB use for their first-class functions). This can take several different formats. For instance, we can give the caller the ability to handle converting from a common delegate to an F# function. If we define a utility like:

```fsharp
[<Qualified>]
type Fun =
  static member Of(act :Action<'T>) = (fun t -> act.Invoke t)
```
  
Then a VB consumer might use:

```
Dim opt = BizLogic.ImportantCalc(42)
If Option.IsSome(opt) Then
  Option.Iterate(Fun.Of(PrintOption), opt)
```
  
However, often I will provide an extension method which handles the conversion internally, saving the consumer a bit of work. For example, a method like:

```fsharp
[<Extension>]
static member IfSome(option, act :Action<'T>) =
  option |> Option.iter (Fun.Of withSome)
```
  
Would turn the previous consumer example into something a bit simpler:

```
Dim opt = BizLogic.ImportantCalc(42)
opt.IfSome(PrintOption)
```

## 5. Do emulate matching with higher-order functions

While C# and VB do not support the rich pattern matching enjoyed in F#, we can still leverage higher-order functions to approximate an expression-oriented API. This technique is especially effective with discriminated unions, as seen here:

```fsharp
[<Extension>]
static member Match(option, withSome :Func<'T,'R>, withNone :Func<'R>) =
  match option with
  | Some value  -> (Fun.Of withSome) value
  | None        -> (Fun.Of withNone) ()
```
  
Given the above definition, consuming C# code might look like this:

```csharp
return ValidationSvc.Validate(input).Match(
  withSome: v  => v.ToString(),
  withNone: () => "<NOT SET>"
);
```

## 6. Prefer overloaded methods to optional parameters

One of the recurring themes of this post is: F# does things differently. And optional parameters are no exception. In F# they are based on the `Option` type. However, in other languages they are handled via dedicated syntax. Due to this mismatch, F# “optional” arguments are, in fact, *required* in a call from C# or VB. In fairness, you *can* achieve the non-F# behavior in F# by careful use of attributes. However, I generally find it easier to explicitly define overloads, which vary in the number of parameters, and thus give callers the effect of having optional arguments.

> ASIDE: a more in-depth look at this topic, by Mauricio Scheffer, may be [found here](http://bugsquash.blogspot.ch/2012/08/optional-parameters-interop-c-f.html).


## 7. Do remember to properly export extension methods

I can’t stress this one enough. In order to properly comply with published specifications — and to support Visual Basic.NET even recognizing an extension method — F# code should decorate each method being provided with `[<Extension>]`. Additionally, any class which contains any such decorated methods should also be decorated with `[<Extension>]`. Finally, somewhere — anywhere — in a library which provides extension methods, you need to add an assembly-level attribute of the form `[<Extension>]`.

> ASIDE: for a more detailed explanation of this mechanism please see [this blog post](http://latkin.org/blog/2014/04/30/f-extension-methods-in-roslyn/), by Lincoln Atkinson.

## 8. Do use classes to provide extension methods

F# actually offers two different ways to define extension methods. The first approach is to decorate a module and one, or more, functions *inside* that module.

```fsharp
[<Extension>]
module Socket =
  [<Extension>]
  let Send (socket :Socket) (frame :byte[]) =
    // ...
 
  [<Extension>]
  let SendAll (socket :Socket) (frame :byte[][]) =
    // ...
```
    
But, as an alternative, you can decorate a class and one, or more, static methods defined on that class. This second approach, besides more closely mirroring the consumer’s mental model, offers a slight advantage: it greatly simplifies the process of providing method overloads. You simply list the separate methods, and implement them as normal.

```fsharp
[<Extension>]
type Socket =
  [<Extension>]
  static member Send(socket :Socket, frame :byte[]) =
    // ...
 
  [<Extension>]
  static member Send(socket :Socket, frames :byte[][]) =
    // ...
```

With the first approach, because each function needs a unique name within the module, you must leverage the `CompiledNameAttribute` to “fake out” method overloads (note: see the next tip for more details).

## 9. Do make judicious use of CompiledNameAttribute

The `CompiledNameAttribute`, like much of F#, is a double-edged sword. It can be used to great effect, especially when trying to support C# or VB. But it can also lead to ruin, increasing complexity and confusion for no real benefit. Use with caution. The concern, you see, is that by using this attribute, you cause a construct to have *two different names*. One will be used by/visible to F#. While the other will be used by/visible to other languages and reflective meta-programming. However, this is sometimes exactly what’s needed. For example, while it often makes sense to collect all of your “language interop” extension methods into a single, dedicated class. For very simple functions, requiring no additional manipulation, it may make sense to avoid the extra layer of indirection. For example, this code:

```fsharp
[<Extension; CompiledName "Send">]
let sendAll (socket :Socket) (msg :byte[][]) =
  // ...
```
  
Would be consumed from F# as:

```fsharp
msg |> Socket.sendAll pub
```

And equally from VB as:

```
pub.Send(msg)
```

Another time where `CompiledNameAttribute` can be helpful in sorting out naming conflicts is when types and modules need to have similar names:

```fsharp
type Result<'T,'Error>
 
[<Qualified; Module(Suffix)>]
module Result =
  // ...
 
[<Extension; CompiledName "Result")>]
type ResultExtensions =
  // ...
```
  
As this example demonstrates, we can partition the functionality around an abstract data type. We can put all the F#-facing components into a module. Then provide the C#-facing equivalents in a class for of static methods. Adding `CompiledName` to the mix ensures a clean, per-language experience.

In F#:

```fsharp
// invoking a function on the Result module
Result.tryCatch (fun () -> getUserFromDb conn)
```

And in C#:

```csharp
// invoking a static method on the ResultExtensions class
Result.TryCatch(() => { return getUserFromDb(conn); });
```

## 10. Prefer records or unions over tuples or primitives

Folks might shy away from exposing F#-centric types like records or discriminated unions to other languages. However, especially if heading the previous guidelines, there’s no reason not to share these powerful constructs with C# and VB. In particular, using types like `Option` (rather than “null checking” or using a `Nullable`) can greatly improve the overall robustness of an API. Consider this example:

```fsharp
let postRequest body :bool =
  // ...
```
  
It can only tell a consumer “Yay!” or “Nay!”. Meanwhile, this example gives calls a much richer experience:

```fsharp
let postRequest body :Result<bool,exn> =
  // ...
```
  
> ASIDE: for more details about this approach to error handling, please see the [excellent work presented here](http://fsharpforfunandprofit.com/rop), by Scott Wlaschin.

## 11. Do expose a LInQ provider (if appropriate)

While not appropriate for all domains, exposing F# functionality via Language-Integrated Query (hereafter, LInQ) can be an excellent bit of “sugar” for C# and VB. Consider this example (leveraging techniques discussed earlier), which tries to combine multiple `Result` instances:

```csharp
var vsn = major.Match(
    m => minor.Match(
        n => revision.Match(
            r => string.Format($"{m}.{n}.{r}"),
            e => e.Message),
        e => e.Message),
    e => e.Message);
 
WriteLine(vsn);
```

Now look at the improvements LInQ offers:

```csharp
var vsn = from m in major
          from n in minor
          from r in revision
          select string.Format($"{m}.{n}.{r}");
 
vsn.IfOK(WriteLine); 
```

It’s true that providing LInQ support will mean defining 3, or more, extension methods against a type. But it’s usually worth the effort.

## 12. Do NOT rely on type aliases

Type aliases can *dramatically* improve the intentionality of your F# code. Unfortunately, they are fully erased at compile time (i.e. stored in special metadata only read by other F# libraries). So a C# or VB consumer won’t see your well-intended, self-documenting type alias… just the raw type underneath.

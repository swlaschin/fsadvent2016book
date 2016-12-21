

# The AsyncArrow #

*All text and code copyright (c) 2016 by Troy Kershaw. Used with permission.*

*Original post dated 2016-12-14 available at https://troykershaw.com/the-asyncarrow/*

**By Troy Kershaw**


'Tis the season for giphing, so for my F# advent post this year we're going to get the top trending gif on giphy.

![That's one excited girl](giphy.gif)

Giphy has a nice little api, all we need to do is send a request, then we'll get a response. Sounds pretty straight forward, huh? In fact, you've probably created a function that takes a request and returns an asynchronous result before, à la `'Req -> Async<'Res>`. Interestingly, in doing so, you've defined the 'Arrow'.

John Hughes, in his paper [Generalising Monads to Arrows](http://www.cse.chalmers.se/~rjmh/Papers/arrows.pdf), describes an arrow as `'a -> M<'b>` where `M` is any monad. We treat this function as a single type until it's time to apply the argument. In fact, you can create a type alias for the async arrow like so:

```fsharp
type AsyncArrow<'a,'b> = 'a -> Async<'b>  
```

I won't be using this type alias for any actual work in this post, but visualising it like this certainly helped me gain an understanding of how they work.

What you'll see in this post is that by representing our async operations as arrows we get monad like properties on the input/output group.

Below is a bit of setup for your `fsx` file if you're following along and running the code. There's a few functions to help us work with `Async` and a few functions in an `AsyncArrow` module. Don't worry about those yet, we'll look at how they work later.

```fsharp
#r "System.Net.Http"

open System.Net.Http  
open System.Text.RegularExpressions  
open System.Diagnostics

// These aliases are just so that I don't wear out my keyboard.
type HttpReq = HttpRequestMessage  
type HttpRes = HttpResponseMessage

// We're calling giphy, remember?
let giphyTrending = "http://api.giphy.com/v1/gifs/trending"

// A few Async functions to make bind and map easy to use
type Async with  
  static member bind (f:'b -> Async<'c>) (a:Async<'b>) : Async<'c> =
    async.Bind(a, f)
  static member map (f:'b -> 'c) (a:Async<'b>) : Async<'c> =
    async.Bind(a, f >> async.Return)

// Don't worry too much about this yet, we'll get to it later
module AsyncArrow =  
  let mapOut (f:'b -> 'c) (a:'a -> Async<'b>) : 'a -> Async<'c> =
    a >> Async.map f

  let mapOutAsync (f:'b -> Async<'c>) (a:'a -> Async<'b>) : 'a -> Async<'c> =
    a >> Async.bind f

  let mapIn (f:'a2 -> 'a) (a:'a -> Async<'b>) : 'a2 -> Async<'b> =
    f >> a

  let after (f:'a * 'b -> _) (g:'a -> Async<'b>) : 'a -> Async<'b> =
    fun a -> g a |> Async.map (fun b -> let _ = f (a,b) in b)
```
    
## I have a little request...

Okay, now that's out of the way, let's write a function that takes a `HttpReq` and returns an `Async<HttpRes>`.

```fsharp
let makeHttpReq : HttpReq -> Async<HttpRes> =  
  fun (req:HttpReq) -> async {
    use client = new HttpClient ()
    return! client.SendAsync req |> Async.AwaitTask }
```
    
See the `HttpReq -> Async<HttpRes>` up there? That's an arrow. By the end of this post you'll hopefully be an arrow fan and you'll start seeing the `'a -> M<'b>` pattern everywhere.

We could have written the function signature with the `AsyncArrow` type alias to be really obvious that we're returning an arrow.

```fsharp
let makeHttpReq : AsyncArrow<HttpReq, HttpRes> = ...  
```

Personally I find `HttpReq -> Async<HttpRes>` easier to read and I use this pattern through the rest of the post.

Back to our makeHttpRequest function. You can run it simply by applying the input argument, in this case a `HttpReq`.

```fsharp
new HttpReq (HttpMethod.Get, giphyTrending)  
|> arrow
|> Async.RunSynchronously
```

That's nice, it works, but I don't care for anything in the `HttpRes`, I just want the content. Let's get that as a string next.

## Map out a route

In our `AsyncArrow` module above we have a `mapOutAsync` function that looks like this:

```fsharp
val mapOutAsync :  
  f:('b -> Async<'c>) -> a:('a -> Async<'b>) -> ('a -> Async<'c>)
```

  
Let's follow the types and see how it works.

If we put our input `a` type and the output type next to each other it becomes obvious how to make it work.

```fsharp
'a -> Async<'b> // this is the input 'a'  
'a -> Async<'c> // this is our output  
```

We can clearly see that a solution is to require a function `'b -> 'c`, and that's exactly what the mapOut function is defined as in the `AsyncArrow` module.

Another solution is to require a function `'b -> Async<'c>`, which is what we need here, and it lines up perfectly with the signature for `mapOutAsync`.

```fsharp
let arrow =  
    makeHttpRequest
    |> AsyncArrow.mapOutAsync (fun res ->
        res.Content.ReadAsStringAsync ()
        |> Async.AwaitTask)
```
        
And we call it simply by giving it a `Req`

```fsharp
new HttpReq (HttpMethod.Get, giphyTrending)  
|> arrow
|> Async.RunSynchronously
```

You can call it the same way as we did above.

## Once we accept our limits, we go beyond them

> - Albert Einstein

Let's be good citizens and add in an `Accept` header.

What we want to do here is create a new arrow that takes an `HttpReq`, adds the `Accept` header to the request and then applies the request to the original arrow, which will return an `Async<HttpRes>`.

The function that does this is called `mapIn`...

```fsharp
val mapIn : f:('a2 -> 'a) -> a:('a -> Async<'b>) -> ('a2 -> Async<'b>)  
```

...and here's how you use it.

```fsharp
let arrow =  
    makeHttpRequest
    |> AsyncArrow.mapOutAsync (fun res ->
        res.Content.ReadAsStringAsync () |> Async.AwaitTask)
    |> AsyncArrow.mapIn (fun (req:HttpReq) ->
        req.Headers.Add ("Accept", "application/json")
        req)
```
        
May your inputs be forever changing

Let's do another `mapIn`, but this time change it's type. It's now time to get rid of that pesky `HttpReq`.

```fsharp
let arrow =  
    makeHttpRequest
    |> AsyncArrow.mapOutAsync (fun res ->
        res.Content.ReadAsStringAsync () |> Async.AwaitTask)
    |> AsyncArrow.mapIn (fun (req:HttpReq) ->
        req.Headers.Add ("Accept", "application/json")
        req)
    |> AsyncArrow.mapIn (fun (url:string) ->
        new HttpRequestMessage(HttpMethod.Get, url))
```
        
We call this by passing in a string url.

```fsharp
giphyTrending  
|> arrow
|> Async.RunSynchronously
```

We're getting errors about authentication, let's add the api key to the request with another `mapIn`.

```fsharp
let arrow =  
    makeHttpRequest
    |> AsyncArrow.mapOutAsync (fun res ->
        res.Content.ReadAsStringAsync () |> Async.AwaitTask)
    |> AsyncArrow.mapIn (fun (req:HttpReq) ->
        req.Headers.Add ("Accept", "application/json")
        req)
    |> AsyncArrow.mapIn (fun (url:string) ->
        new HttpRequestMessage(HttpMethod.Get, url))
    |> AsyncArrow.mapIn (fun url -> url + "?api_key=dc6zaTOxFJmzC")
```
    
Run that and you'll get a lovely json response.

## The home stretch

Let's get the first url from the list using `mapOut`.

```fsharp
let arrow =  
    makeHttpRequest
    |> AsyncArrow.mapOutAsync (fun res ->
        res.Content.ReadAsStringAsync () |> Async.AwaitTask)
    |> AsyncArrow.mapIn (fun (req:HttpReq) ->
        req.Headers.Add ("Accept", "application/json")
        req)
    |> AsyncArrow.mapIn (fun (url:string) ->
        new HttpRequestMessage(HttpMethod.Get, url))
    |> AsyncArrow.mapIn (fun url -> url + "?api_key=dc6zaTOxFJmzC")
    |> AsyncArrow.mapOut (fun res ->
        Regex.Match(res, "\"url\":\"(.+?)\"").Groups.[1].Value.Replace ("\\", ""))
```
        
Sorry about the regex, I didn't want to add a dependency on a Json parser.

## After all is said and done...

Let's open the gif in your browser.

```fsharp
let arrow =  
    makeHttpRequest
    |> AsyncArrow.mapOutAsync (fun res ->
        res.Content.ReadAsStringAsync () |> Async.AwaitTask)
    |> AsyncArrow.mapIn (fun (req:HttpReq) ->
        req.Headers.Add ("Accept", "application/json")
        req)
    |> AsyncArrow.mapIn (fun (url:string) ->
        new HttpRequestMessage(HttpMethod.Get, url))
    |> AsyncArrow.mapIn (fun url -> url + "?api_key=dc6zaTOxFJmzC")
    |> AsyncArrow.mapOut (fun res ->
        Regex.Match(res, "\"url\":\"(.+?)\"").Groups.[1].Value.Replace ("\\", ""))
    |> AsyncArrow.after (fun (input, output) ->
        Process.Start output)
```

What is really nice about the after function is that you get the output, but also the input that caused the output. Very helpful for logging.

## Next steps

**Conferences**

I'll be speaking about the AsyncArrow at [NDC London](http://ndc-london.com/), 16 - 20 January 2017, so come and listen and say hi.

**Projects**

[Finagle](https://github.com/twitter/finagle) by Twitter 
While different to our use in this post, the `'Req -> Async<'Res>` signature implements a server.

Jet is planning to open source our `AsyncArrow` module. We're also discussing naming and there's a chance it will be released as `AsyncFunc`, so look out for that too.

**Papers**

* [Your Server as a Function](https://monkey.org/~marius/funsrv.pdf) by Marius Eriksen 
* [Programming with Arrows](http://www.cse.chalmers.se/~rjmh/afp-arrows.pdf) by John Hughes 
* [Generalising Monads to Arrows](http://www.cse.chalmers.se/~rjmh/Papers/arrows.pdf) by John Hughes (referenced in the post above)

**People**

I was introduced to arrows by [Leo Gorodinski](https://twitter.com/eulerfx), and he said you could reach out to him. 

Feel free to [reach out to me as well](https://twitter.com/troykershaw).

----

Fun, huh?

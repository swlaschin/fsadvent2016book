
# An Introduction To Testing Suave WebParts #

*All text and code copyright (c) 2016 by Matt Olson. Used with permission.*

*Original post dated 2016-12-11 available at https://mattjolson.github.io/2016/12/12/testing-suave-webparts.html*

**By Matt Olson**

One of the joys of working with [Suave](http://suave.io/) is the testability that falls out of its composition model. To make those tests really sing, I’ve built a few helpers that wrap some of the underlying type complexity. This in turn allows my tests to be more concise, focused on the behaviour I’m building.

None of this is terribly complex, and that’s part of the fun.

## A Brief Introduction to WebParts

Suave’s routing logic is specified by composing WebParts, which are functions on `HttpContexts`:

```fsharp
type WebPart = HttpContext -> Async<HttpContext option>
```

Briefly, a WebPart takes an `HttpContext` as input, does something with it, and returns an `option` over `HttpContext` – either `Some a`, where `a` is whatever the WebPart was intended to do, or `None`, indicating that the WebPart doesn’t care about this particular input. All of this is wrapped up in an Async workflow, which we don’t need to worry about just now.

What’s important here is that `HttpContext` contains all the properties we care about when testing our routing logic. We’re going to build a bunch of tools that operate on `HttpContexts`, specifically focused on testing this logic.

For a deeper look at Suave routing, [have a look at the docs](https://suave.io/routing.html).

## The Very Basics - Requests and Responses

So let’s start by building a request, throwing it at our router, and verifying that we get a 200 OK out the other side. We’ll fill in some basic request parameters on `HttpContext.empty`:

```fsharp
let GetRequest u =
    let uri = new System.Uri("http://some.random.tld" + u)
    let rawQuery = uri.Query.TrimStart('?')
    let req = { HttpRequest.empty with url = uri ;
                ``method`` = HttpMethod.GET ;
                rawQuery = rawQuery }
    { HttpContext.empty with request = req }
```
    
This is perfectly usable as-is, but I’m just noticing that we can separate some concerns by splitting the method and URI parts, and calling them separately:

```fsharp
let WithUri u hc =
    let uri = new System.Uri("http://some.random.tld" + u)
    let rawQuery = uri.Query.TrimStart('?')
    let req = { hc.request with url = uri ;
                rawQuery = rawQuery }
    { hc with request = req }

let AsGetRequest hc =
    let req = { hc.request with ``method`` = HttpMethod.GET }
    { hc with request = req }

let GetRequest u =
    HttpContext.empty
    |> WithUri u
    |> AsGetRequest
```

This shows off our general pattern for building test data: We’re writing combinators on `HttpContext`, adding info as we need to. We can use the same approach to add headers, fill in a request body, set cookies, and so on. For now we’ll proceed with what we have.

Once we’ve done that, we can throw it at a WebPart and see what we get:

```fsharp
[<Test>]
let ``resource/ with an integer parameter returns 200 OK`` () =
    GetRequest "/resource/1337" 
    |> mySuaveRouter
    |> ...?
```
    
(I’m using NUnit and [FsUnit](https://fsprojects.github.io/FsUnit/), but that’s more or less irrelevant.)

So we threw in an `HttpContext` with a well-populated request, and we’d like to examine the result... but we need to unwrap it first. This part isn’t as much fun as building a set of combinators, but it’s still pretty straightforward:

```fsharp
let expectSome hc =
    match hc with
    | Some a -> a
    | None   -> failwith "Expected 'Some'"

let extractContext aohc =
    aohc |> Async.RunSynchronously |> expectSome

let extractStatus hc = hc.response.status

...

[<Test>]
let ``resource/ with an integer parameter returns 200 OK`` () =
    GetRequest "/resource/1337" 
    |> mySuaveRouter
    |> extractContext
    |> extractStatus
    |> should equal HTTP_200
```
    
Not as much theoretical fun as chaining combinators, but it does the job. Note that `failwith` in `expectSome` – if we get a `None` out of our router, that thrown exception will have NUnit fail our test.

Notice what we’re *not* doing: We’re not firing up an OWIN or Nancy instance, installing an ASP.NET site, configuring a DI container so it knows how to build controllers, or any other overhead. All we’re doing is calling a function!

## Knowing Where to Look

Let’s try something else. Maybe our GET resource is supposed to return a JSON document with the resource ID in it, and we want to do a simple string match to verify that. We’ll need a way to get the response body, which means destructuring another internal type:

```fsharp
let contentAsString hc = 
    match hc.result.content with
    | Bytes b -> b |> System.Text.Encoding.Default.GetString
    | NullContent -> ""
    | SocketTask st -> "OMG a socket"

[<Test>]
let ``resource/ returns the integer ID of the requested resource`` () =
    GetRequest "/resource/1337"
    |> mySuaveRouter
    |> extractContext
    |> contentAsString
    |> should haveSubstring ``"id": 1337``
```
    
Nothing particularly surprising there. As with `GetRequest`, we’re reaching into the `HttpContext` for the data we need, this time pulling out bits of the result. Because the `HttpContext` object tracks the state of the whole web call, it has a lot of structure to it. If you’re used to dealing with `IHttpActionResults`, your standard approach might be combining educated guesses with autocompletion, but that’s likely to be less effective here.

A lot of the Suave testing story involves digging into the types in HttpContext and figuring out what needs to be set (or matched on), where. For that, you’ll want to keep [Http.fs](https://github.com/SuaveIO/suave/blob/master/src/Suave/Http.fs) open in a browser tab (or maybe an IDE).

## A Word About Routers

You’ll notice that I’m calling the router with a single argument (as I ought to, since `WebParts` are defined on a single argument). Obviously, any meaningful webservice is going to have external dependencies – persistence stores, monitoring integrations, that sort of thing. I don’t want to spin those up every time I run a test, so what gives?

Basically, I can define that router as a partial function that takes those dependencies as parameters, and stub in whatever I need for testing. So, something like:

```fsharp
let lookupSucceeds (l : Suave.Logging.Logger) (n : int) =
    Resource.empty |> Some
let fakeLogger = { new Suave.Logging.Logger with
    member l.Log level logfn = ()
}

let mySuaveRouter = myUnderlyingRouter fakeLogger lookupSucceeds
```

This lets me test failure cases, too:

```fsharp
let lookupFails (l : Suave.Logging.Logger) (n : int) = None

[<Test>]
let ``resource/ returns 404 if the resource can't be found`` () =
    GetRequest "/resource/1337" 
    |> (myUnderlyingRouter fakeLogger lookupFails)
    |> extractContext
    |> extractStatus
    |> should equal HTTP_404
```
    
This is just dependency injection, only we’re doing it without a container. The ease with which F# lets us pass around partially-applied functions makes a DI container unnecessary.

## Wrapping Up

This has been a high-level look at testing Suave’s routing logic. There’s certainly more to say – we could use [Aether](https://xyncro.tech/aether/guides/) for deeper inspection and manipulation of `HttpContexts`, look at the advantages of testing an HTTP resource’s routing at the top level versus testing smaller composed units, and so on – but this is the workflow I’ve stumbled upon and made work for production code. I hope you’ve found it helpful.

If/when you notice something I can improve, please let me know about it on Twitter.
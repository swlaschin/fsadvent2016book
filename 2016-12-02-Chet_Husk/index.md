
# A Host of Issues: Migrating from UserVoice To GitHub Issues with Canopy and OctoKit.net #

*All text and code copyright (c) 2016 by Chet Husk. Used with permission.*

*Original post dated 2016-12-02 available at https://chethusk.blogspot.co.uk/2016/12/a-host-of-issues-migrating-from.html*

**By Chet Husk**


On October 6th, [Krzysztof Cieslak](https://twitter.com/k_cieslak) had had enough. He'd grown tired of spending time working on submissions
to UserVoice for F# language enhancements and after some prodding submitted [THE SUGGESTION THAT WOULD LIVE IN INFAMY](https://fslang.uservoice.com/forums/245727-f-language/suggestions/16529041-close-user-voice-in-favor-of-github-issues).
It immediately generated some frenzy, and I'd had a bad day at work, so on a whim I decided to finally learn [Canopy](https://lefthandedgoat.github.io/canopy/),
a Selenium library for F#, and try my hand at a first pass of a migration of issues from UserVoice to GiHub Issues.

Canopy is really a fantastic library.  In essence it's a tightly-crafted DSL on top of the Selenium WebDriver library, which allows for easier automation of web browsers though code.  It's most commonly used for UI acceptance tests in my experience, but we're going to pervert its intentions a bit.

But first, some links:
* The old uservoice is here: https://fslang.uservoice.com/forums/245727-f-language
* The new language design repo is here: https://github.com/fsharp/fslang-suggestions
* The repo for the code that ended up generating the new language design repo is here: https://github.com/dsyme/fsharp-lang-suggestions-scripts

And please don't judge to code quality too harshly, [Jared](https://twitter.com/cloudroutine) and I were kinda blazing through this thing in off hours!

So really what we had to accomplish was three things:

1. Discover the list of all issues on UserVoice
1. Pull over metadata for each of the issues:
  * Submitter
  * Content
  * Date
  * Votes
  * Comments
  * Submitter
  * Date
  * Official Responses
  * Responder
  * Date
  * Content
1. Use that metadata to create a matching issue on GitHub

So to start with, after hashing out some requirements we landed with the following set of base models:

```fsharp
type Comment = 
    {   Submitter : string
        Submitted : DateTime
        Content : string }
    
type Response = 
    {   Responded : DateTime
        Text : string
        Exists : bool }

type Idea = 
    {   Number : string
        Submitter : string
        Submitted : DateTime
        Title : string
        Text : string
        Votes : int32
        Comments : Comment list
        Status : string 
        Response : Response } 
```

With these models in hand, we can now go parse sections of the each page out to get each `Idea`.
All of the code directly related to scraping the pages is [here](https://github.com/dsyme/fsharp-lang-suggestions-scripts/blob/master/src/Scraping.fs).
An interesting point is that there's no master list of all issues for a forum, you've got to go scrape through each different category of issues to get the entire list,
so that's what we've done in the `discoverIdeas` function.

```fsharp
// get a link from an element
let href = attr "href"
let userVoiceItemClass = ".uvIdeaTitle a" // ideas are an anchor inside the uvIdeaTitle
// this is the link to the next page, if there is one
let nextpageClass = "a.next_page"
let discoverIdeas () = 
    let rec parseUrlsFromPage address =
        // navigate to the address 
        url address
        // all the links from the elements that match the uservoice item matcher
        let pageItemLinks = elements userVoiceItemClass |> List.map href

        // if there is a next button, traverse it and collect the links from the next page
        match someElement nextpageClass |> Option.map href with
        | None -> pageItemLinks
        | Some next -> pageItemLinks @ parseUrlsFromPage next

    let pages = 
        [
            "https://fslang.uservoice.com/forums/245727-f-language"
            "https://fslang.uservoice.com/forums/245727-f-language/status/1225913"
            "https://fslang.uservoice.com/forums/245727-f-language/status/1225914"
            "https://fslang.uservoice.com/forums/245727-f-language/status/1225915"
            "https://fslang.uservoice.com/forums/245727-f-language/status/1225916"
            "https://fslang.uservoice.com/forums/245727-f-language/status/1225917"
        ]

    pages |> List.collect parseUrlsFromPage |> List.distinct
```

Next was the grunt work of cycling through each of the links we just found and parsing them.  This is a hairy chunk, but relatively straightforward:

```fsharp
/// assumes that this is a 'time' html element
let time (el : IWebElement) = el |> attr "datetime" |> DateTime.Parse

/// used to find things that can be turned into markdown links
let markdownLinkRegex = Regex("(\/ideas\/(\S+))\s*")

/// attempts to rewrite all urls to uservoice to map to the matching issue document on github.
let rewriteUrls (text : string) =
    // simple match and replace
    let urls = [
        "https://fslang.uservoice.com/forums/245727-f-language/suggestions/"
        "http://fslang.uservoice.com/forums/245727-f-language/suggestions/"
    ] 
    let rewritten = urls |> List.fold (fun (s:string) url -> s.Replace(url, "/archive/suggestion-")) text
    
    // now rewrite things into markdown link syntax : [XXX](YYY)
    match markdownLinkRegex.Matches(rewritten) with
    | m when m.Count = 0 -> rewritten 
    | m -> 
        m 
        |> Seq.cast<Match>
        |> Seq.fold (fun s m' -> 
            // rewrite the /ideas/suggestion string to [name of link](/ideas/suggestion-XXXX)
            let whole = m'.Groups.[0].Value.Trim()
            let file = m'.Groups.[1].Value.Trim()
            s.Replace(whole, sprintf "[%s](%s.md)" whole file)
         ) rewritten        

/// this is a recursive call because the comments themselves are paged and so we need to be able to 
/// concat a list of comments across pages that we visit
let rec parseCommentsFromPage address = 
    let parseComment pos (el :IWebElement) : Comment =
        let submitter = el |> elementWithin "span" |> read
        let submitted = el |> elementWithin "time" |> read |> DateTime.Parse
        let content = el |> elementWithin "div.typeset" |> read
        {Submitter = submitter; Submitted = submitted; Content = content |> rewriteUrls }

    url address
    let commentBlocks = unreliableElements ".uvIdeaComments li.uvListItem" |> List.mapi parseComment
    match someElement "a.next_page" |> Option.map href with
    | None -> commentBlocks
    | Some page -> commentBlocks @ parseCommentsFromPage page

/// this is mostly just a lot of pulling out individual fields by accessor classes
let parseIdeaFromPage pos address : Choice<Idea,string> =
    url address
    printfn "parsing idea %d: %s" pos address
    try
        let voteCount = element "div.uvIdeaVoteCount" 
        let number = voteCount |> attr "data-id"
        let votes = voteCount |> elementWithin "strong" |> read |> Int32.Parse
        let title = element "h1.uvIdeaTitle" |> read
        let submitter = element "div.uvUserActionHeader span.fn" |> read 
        
        // the text of a comment can have uservoice links, so lets rewrite those to refer to relative urls so that links in 
        // the new github issues are valid still! 
        let text = defaultArg (someElement "div.uvIdeaDescription div.typeset" |> Option.map (read >> rewriteUrls)) ""
        let submitted = element "section.uvIdeaSuggestors div.uvUserAction div.uvUserActionHeader span time" |> time
        
        let comments = parseCommentsFromPage address |> List.rev
        let state = 
            someElement "span.uvStyle-status" 
            |> Option.map (fun el ->((attr "class" el)
                                        .Split([|' '|], StringSplitOptions.RemoveEmptyEntries) |> Array.last)
                                        .Substring "uvStyle-status-".Length)

        let response = 
            let responded = someElement "article.uvUserAction-admin-response time" |> Option.map time 
            // likewise, responses can have links that need rewritten
            let text = someElement "article.uvUserAction-admin-response .typeset" |> Option.map (read >> rewriteUrls)
            match responded, text with
            | Some r, Some t -> { Responded = r; Text = t; Exists = true }
            | _, _ -> {Exists = false; Responded = DateTime.MinValue; Text = ""}


        { Number = number
          Submitter = submitter
          Submitted = submitted
          Title = title
          Votes = votes
          Text = text
          Comments = comments
          Status = defaultArg state "open"
          Response = response } : Idea |> Choice1Of2
    with
    | ex -> Choice2Of2 <| sprintf "error accessing %s: %s" address (ex.Message)
```

After everything was parsed, we did a quick Json.Net dump of the Idea list into a file so we wouldn't have to scrape that again.
It took about 20 minutes to scrape all of UserVoice because Canopy seems to implicitly have a single webdriver context.  I thought about parallelizing by looking into passing contexts around to use, but I couldn't find a way to do that and once we had the data everything else fell into place.

Having secured the data, we then needed to make markdown issue templates to render the ideas into a form that would look good on the GitHub Issue.
My preferred templating library in .Net is [DotLiquid](http://dotliquidmarkup.org/) for its simple setup and reliability.  If you'd like to see what the template looks like,
take a quick detour and check it out [here](https://github.com/dsyme/fsharp-lang-suggestions-scripts/blob/master/templates/_idea_submission.liquid). Simple, no?  I did have to do a bit of configuration to get the templating system to recognize F# records, but luckily that work had already been done by (I think) Tomas Petricek and Henrik Feldt. Basically what we have to do to use the templates in a nice way is register the public members of any type we want to use as a model in the template, and there are some special cases around Seq, List, and Option that have to be handled for dotLiquid to work. The code looks like this:

```fsharp
module FslangMigration.Templating

open DotLiquid
open FSharp.Reflection
open System.IO
open System.Text.RegularExpressions
open System.Collections.Generic

// we want to store type registrations in DotLiquid so that we don't have to recompute them
let registrations = Dictionary<_,_>()

let parseTemplate<'T> template =
    // walk types and register them
    let rec registerTypeTree ty =
        if registrations.ContainsKey ty then ()
        elif FSharpType.IsRecord ty then
            // register all the fields of the record
            let fields = FSharpType.GetRecordFields ty
            Template.RegisterSafeType(ty, [| for f in fields -> f.Name |])
            registrations.[ty] <- true
            for f in fields do registerTypeTree f.PropertyType
        elif ty.IsGenericType then
            // certain generic types have to be handled differently
            let t = ty.GetGenericTypeDefinition()
            if t = typedefof<seq<_>> || t = typedefof<list<_>>  then
                // collections are supported and just need the inner items type registered
                registrations.[ty] <- true
                registerTypeTree (ty.GetGenericArguments().[0])     
            elif t = typedefof<option<_>> then
                // for option, we manually register some properties
                Template.RegisterSafeType(ty, [|"Value"; "IsSome"; "IsNone";|])
                registrations.[ty] <- true
                // and then the inner type
                registerTypeTree (ty.GetGenericArguments().[0])
            elif ty.IsArray then          
                registrations.[ty] <- true
                // arrays just have a shortcut due to the .Net reflection API
                registerTypeTree (ty.GetElementType())
        else ()
    
    // register the model type
    registerTypeTree typeof<'T>
    // parse the template
    let t = Template.Parse template
    // given an label name and an instance of the model, render the template with a dictionary made of all of the properties of the model
    fun k (v:'T) -> t.Render(Hash.FromDictionary(dict [k, box v]))


Template.NamingConvention <- NamingConventions.CSharpNamingConvention()

//host the templates from the file system
let templatedir = Path.GetFullPath "../templates/"
let fs = DotLiquid.FileSystems.LocalFileSystem templatedir
Template.FileSystem <- fs :> DotLiquid.FileSystems.IFileSystem

// this helper functions ensures that we can never use the wrong template for the wrong model type
let templateFor<'a> file variableName = parseTemplate<'a> (File.ReadAllText(Path.Combine(templatedir, file))) variableName

let archiveTemplate    = templateFor<Idea>     "_idea_archive.liquid"    "idea"
let submissionTemplate = templateFor<Idea>     "_idea_submission.liquid" "idea"
```

Finally came the hard part - interacting with the GitHub api.  Thankfully, we had OctoKit.net available to smooth over calling the api directly, but as everyone knows the GitHub api is a bit aggressive when it comes to rate limiting consumers. You're limited to 20 calls per minute, and a daily allotment of 5000 calls overall.  The initial version of the upload code was incredibly parallel, queuing up all the work making use of Asyncs all over the place.  That code bombed almost immediately due to burning through our api allowance.  We eventually settled on including delays periodically to slow down the overall rate of upload, but the rate limit made the upload process crawl to hours instead of what we had hoped would be minutes.  At this moment there are two separate attempts to manage throttling more intelligently in the Github.fs file, however none were good enough.  In fact, months after we did this work, I've started a MailboxProcessor-based version that would handle caps on my fork of the repo but it's still not up to par.  So if anyone has any pointers to intelligently manage rate limiting in a general/intelligent way that changes based on output of the api client after each call, let me know!

The code for uploading to GitHub isn't really complicated at all, and lives [here](https://github.com/dsyme/fsharp-lang-suggestions-scripts/blob/master/src/Github.fs).
It's mostly a straightforward mapping of `Idea -> Issue GitHub Model` and then we POST that model up.  We do do a few interesting things like assign labels and close the issue automatically if the issue was closed/rejected/etc on UserVoice, as one would expect.

Overall, I'd say that two people spent ~10 hours each working on this, and A LOT of that time was waiting on uploads to break.  We spent some time tweaking the issue templates for formatting, but that was a very iterative process.  Canopy made it incredibly simple to grab the data we needed to do the migration, DotLiquid made it easy to render nice markdown for the Issues, and OctoKit...OctoKit wasn't the worst.

My overall hope with this post is to show that F# is as great for relatively simple, one-off tasks as it is for more in-depth projects like dependency management, finance, microservices, and any number of other realms (though I do use it there as well). So I encourage everyone who's looking for ways to include it at work to try and tackle your next annoying problem with a fsx script. That's how this project started, until it grew too unwieldy.

Happy Holidays, and Merry F#ing!
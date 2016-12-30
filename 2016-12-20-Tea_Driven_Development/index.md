


# Feeding On F# #

*All text and code copyright (c) 2016 by Tea Drinking Development. Used with permission.*

*Original post dated 2016-12-29 available at https://teadrivendev.github.io/2016/12/29/FsAdvent-Rss-Feed/*

**By Tea Drinking Development**


## Vorgeplänkel

First, I need to apologize for being so late with my contribution. I had actually booked a slot on December 20th, but my originally chosen topic turned out to be a very bad fit for a written article (I'd like to do that as a video at some point, though), so I had to start over with something else, which also proved very difficult to write for some reason, but here we go....

Chances are that you've come to this article from the 2016 F# Advent Calendar overview on Sergey Tihon's blog, where the full schedule is presented in a table showing the already written articles as well as the dates and authors for the remaining ones, so you get a full overview of the whole event.

One thing that this doesn't facilitate (as inherent in the medium) is keeping track of which articles you have or haven't read. What would be nice for that is an RSS feed, but we don't have that. We have all the information that we need, though, to build one ourselves - using F#, of course.

This will be a rather low-tech solution - we run a script that generates the XML file for the RSS feed, and then we put that somewhere on the Internet. That means there is some manual work involved once a day or so, but that way we only need somewhere to put a static file (I'm going to use GitHub Pages). This manual step can be eliminated if there's a place available to run a small webservice that checks the overview a few times a day and regenerates the feed on the server as needed.

## Reading

We will use the HTML type provider from `FSharp.Data` to deal with the HTML. So we add the FSharp.Data package with Paket, reference the DLL in a new `.fsx` script file and create our `Page` type from the type provider.

```fsharp
#r @"..\packages\FSharp.Data\lib\net40\FSharp.Data.dll"

open FSharp.Data

[<Literal>]
let OverviewUrl = @"https://sergeytihon.wordpress.com/2016/10/23/f-advent-calendar-in-english-2016/"

type Page = HtmlProvider<OverviewUrl>
```

Note that this does not constrain us to processing that one specific page. As long as the structure is what we expect, we can still pass in a different page later.

If we look at the HTML for a single post, that is a single row in the table, we find this:

```html
<tr>
  <td>Dec 15 (Thu)</td>
  <td><a href="https://twitter.com/ReedCopsey/status/790203670637912064">Reed Copsey, Jr.</a></td>
  <td><a href="http://reedcopsey.com/2016/12/15/christmas-trees-in-wpf-2016-edition/">Christmas Trees in WPF, 2016 Edition </a></td>
</tr>
```

What we need from this is the date, the author's name, the post title and the post URL, all of which are easy enough to obtain. We'll keep them in this small F# record:

```fsharp
type Entry = { Date : string; Author : string; Title : string; Url : string }
```

However, before we start implementing this, we'll have a look at a special case from 2015:

```html
<tr>
  <td> Dec 05 (Saturday)</td>
  <td> <a href="https://twitter.com/ScottWlaschin/status/658388060032401412">Scott Wlaschin</a></td>
  <td>
    <a href="http://fsharpforfunandprofit.com/posts/13-ways-of-looking-at-a-turtle/">Thirteen ways of looking at a turtle</a><br />
    <a href="http://fsharpforfunandprofit.com/posts/13-ways-of-looking-at-a-turtle-2/">Thirteen ways of looking at a turtle (part 2)</a><br />
    <a href="http://fsharpforfunandprofit.com/posts/13-ways-of-looking-at-a-turtle-3/">Thirteen ways of looking at a turtle &#8211; addendum</a>
  </td>
</tr>
```

Scott had so much content (as usual, one might say) that a single post would have gotten far, far too long, so he published several installments, all of which are listed for his one calendar entry. The proper way to handle that in our feed is probably creating an entry for each installment, so we need to be able to extract all the links and then treat them as if they were separate rows.

There are also rows in the table that we want to ignore - the header row - entry rows without actual links to articles, either because they're simply in the future, and the articles don't exist yet, or because they contain a "coming soon" notice, like the one for this post at the time of writing

If we look at a row, it's always a `<tr>` node containing three `<td>` nodes, the first of which is the date as simple text, the second the author's name in a link to the tweet or message announcing their participation, and the third contains the article links as `<a>` tags, if there are any.

A delay notice in the third column starts with a `<span>` tag to change the text color:

```html
<tr>
  <td>Dec 20 (Tue)</td>
  <td><a href="https://twitter.com/TeaDrivenDev/status/790270967356555268">Тэ дрэвэт утвикλэрэн</a></td>
  <td><span style="color:#ff6600;">slightly delayed, coming <a href="https://twitter.com/TeaDrivenDev/status/811208426562945029">soon</a>&#8230;</span></td>
</tr>
```

We'll read the content of the cells with Active Patterns, because F# programmers **love** Active Patterns (I know I do).

![](twitter1.gif)

And they actually make things nice and clear in this case, as we'll see.

```fsharp
// HtmlNode.InnerText() returns the visible text of a node, ignoring any "meta" information like
// links, so we can use this to extract both the date and the author's name.
let (|TextValue|) (n : HtmlNode) = n.InnerText().Trim()

// If the <td> node's first descendant is a <span>, this is a delay notice, so there are no
// articles here. Otherwise we'll assume there will be 1..n links to articles in this cell,
// given in the normal HTML way in <a> tags, from which we'll need the "href" attribute and the
// InnerText respectively.
let (|Posts|) (n : HtmlNode) =
    match n.Descendants() |> Seq.tryHead with
    | Some node when node.Name() = "span" -> Seq.empty
    | _ ->
        n.Descendants("a")
        |> Seq.map (fun link -> link.Attribute("href").Value(), link.InnerText())

// This creates an Entry record with the date and author from the first two columns for each
// article found in the third column. All the non-article table rows are implicitly ignored here
// because there are no link/title tuples returned for them.
let parseRow (row : HtmlNode) =
    match row.Descendants("td") |> Seq.toList with
    | [ TextValue date; TextValue name; Posts posts ] ->
        posts
        |> Seq.map (fun (url, title) ->
            { Date = date.Substring(0, 6); Author = name; Title = title; Url = url })
    | _ -> Seq.empty

// A few small helper functions to make processing the HTML more readable
let descendantsNamed name (node : HtmlNode) =
    node |> HtmlNode.descendants false (fun n -> n.HasName name)

let firstDescendantNamed name node =
    descendantsNamed name node |> Seq.head

let innerText (node : HtmlNode) = node.InnerText()

// In order to be able to easily use this script for processing the calendar overview pages for
// different years, we also need to extract the year.
let getYear body =
    let title = body |> firstDescendantNamed "article" |> firstDescendantNamed "h1" |> innerText

    let xmatch = System.Text.RegularExpressions.Regex.Match(title, @"\d{4}$")
    xmatch.Captures.[0].Value |> int

// This creates the Entry records for the whole HTML page. Note that we use the Page type we've
// created  above, but load the HTML file that we specify here, so we can just as well use this to
// parse the page for 2015's advent calendar.
let getEntries body =
    body
    |> firstDescendantNamed "table"
    |> firstDescendantNamed "tbody"
    |> descendantsNamed "tr"
    |> Seq.collect parseRow

let getValues (pageUrl : string) =
    let body = Page.Load(pageUrl).Html.Body()

    getYear body, getEntries body
```
    
Let's look what that gets us.

```fsharp
let year, entries =
    let year, entries = getValues OverviewUrl

    year, entries |> Seq.toList
```
    
## Building

Now that we have all the information, we can reassemble it into the RSS feed.

The basic XML for an RSS feed looks like this:

```
<rss version="2.0">
  <channel>
    <title><!-- The title to show in RSS readers --></title>
    <link><!-- The address of the site the feed is for, e.g. the home page of a news website --></link>
    <description><!-- A potentially longer description --></description>
    <pubDate><!-- The original publication date --></pubDate>
    <lastBuildDate><!-- The date the feed was last modified --></lastBuildDate>
    <language><!-- The content language, of course --></language>
  </channel>
</rss>
```

Inside the `channel` tag, there will be the individual items, each looking like this:

```
<item>
  <title><!-- The title to be displayed in readers --></title>
  <link><!-- The link to the actual content --></link>
  <guid><!-- A globally unique ID, apparently needs to be a URL --></guid>
  <pubDate><!-- The publication date --></pubDate>
</item>
```

We could use format strings here, but as I don't feel like dealing with multiline strings, it's going to be XLINQ.

```fsharp
#r "System.Xml.Linq"

open System
open System.Linq
open System.Xml.Linq


let xn name = XName.Get name

// As we will get the unencoded characters for apostrophes etc. from the HTML, we'll use this small
// function to make sure we have our text properly encoded for use in XML.
let xmlEncode s =
    let node = Xml.XmlDocument().CreateElement "root"
    node.InnerText <- s
    node.InnerXml

// As there are sometimes posts on January 1st of the new year, we need to make sure they get the
// correct publication date as well.
let getEntryYear baseYear (date : string) =
    match date.Split(' ') |> Array.head with
    | "Jan" -> baseYear + 1
    | _ -> baseYear

// Dates in RSS must be in RFC 1123 format
let encodeDate (date : DateTime) = date.ToString "r"

let entryXml baseYear entry =
    XElement(xn "item",
        XElement(xn "title", sprintf "%s | %s | %s" entry.Date (xmlEncode entry.Author) (xmlEncode entry.Title)),
        XElement(xn "link", entry.Url),
        XElement(xn "guid", entry.Url),
        XElement(xn "pubDate", sprintf "%s %i" entry.Date (getEntryYear baseYear entry.Date) |> DateTime.Parse |> encodeDate))


let emptyFeed year =
    let xe =
        let date = encodeDate DateTime.Now

        let title = sprintf "F# Advent Calendar %i" year

        XElement(xn "rss",
            XAttribute(xn "version", "2.0"),
            XElement(xn "channel",
                XElement(xn "title", title),
                XElement(xn "link", OverviewUrl),
                XElement(xn "description",  title),
                XElement(xn "pubDate", date),
                XElement(xn "lastBuildDate", date),
                XElement(xn "language", "en")))

    xe

let addEntries (feed : XElement) (entries : XElement seq) =
    entries
    |> Seq.toArray
    |> feed.Descendants(xn "channel").First().Add

    feed

let createFeed url =
    let year, entries = getValues url

    year,
    entries
    |> Seq.rev // put the latest entry first in the feed
    |> Seq.map (entryXml year)
    |> addEntries (emptyFeed year)
```
    
## Writing

That was all pretty straightforward. Now that we can create the feed XML, we only need to write the actual XML file to disk.

```fsharp
open System.IO

let repositoryDir = @"D:\Development\Projects\Active\TeaDrivenDev.github.io"

let rssFile = @"public\fsadvent%i.rss"

let writeFeed fileName feed =
    File.WriteAllText(fileName, string feed)

OverviewUrl
|> createFeed
|> (fun (year, feed) -> writeFeed (Path.Combine(repositoryDir, sprintf (Printf.StringFormat<_> rssFile) year)) feed)
```

Now that we have created the RSS file(s), the only thing left is publishing it. Depending on where we do that, it might require different steps, but for the typical things like FTP upload or publishing via Git, FAKE makes that easy anyway. I'm putting the feed on my GitHub Pages site, so we need to commit it to Git. We use Paket to add FakeLib to our project, then reference it in our script, and add the calls to stage and commit the RSS file(s). We could also push to GitHub from here, but I prefer to do that by hand.

```fsharp
#r @"..\packages\FAKE\tools\FakeLib.dll"

open Fake.Git.CommandHelper

gitCommand repositoryDir "add *.rss"
gitCommand repositoryDir "commit -m \"Updated F# advent calendar RSS feed\""
```

## Not Dea.... Done Yet

That's it, essentially. For the (in this case short remaining) duration of the F# Advent Calendar, we run this once a day or so and publish the updated feed to GitHub.

This script can process the overview pages for 2015 and 2016; the one for 2014 is a little different and would need special treatment.

There is one very annoying problem that we are going to have with the feed specifically if (as with me) Feedly is being used as the RSS aggregator. When subscribing to a feed, Feedly only pulls the first 10 items (which in our case are the latest ones, since we've reversed the order). If we're adding feed with over 50 entries at once now, we'd lose most of the articles. There is a workaround, though, but it requires control over publishing the feed: *Updating* a feed in Feedly appears to have no limit on the number of items. That means if we initially create the feed with only 10 items, publish it and add it to Feedly, and only *then* generate the rest of the items and publish again, Feedly will correctly pick up all the entries. So for the first time creating the feed, we need a slightly modified version of the `createFeed` function:

```fsharp
let createFeed' url =
    let year, entries = getValues url

    entries
    |> Seq.truncate 10
    |> Seq.rev
    |> Seq.map (entryXml year)
    |> addEntries (emptyFeed year)
```
    
For all subsequent generation runs, we switch back to the original function without the `Seq.truncate` call.

There's still one issue with that solution that I haven't been able to fix, though: Feedly will only give those first 10 entries their actual `pubDate` values from the RSS file, and date all others as they arrive. That means that in the case of 2016, we'll get correctly dated entries until about the beginning of December, and after that, there will be 40+ articles with the date when we first publish the full feed. But as we have the date in the title for each entry, and the specific dates generally shouldn't have any relevance for the articles' content, that's probably something we can live with.

## Possible improvements

* Make this a webservice as mentioned at the beginning, to get rid of the manual updating
* Get the actual publication dates and descriptions/content for the individual posts. That would require following the link to each post, getting the RSS feed for the respective blog, finding the entry for the post and more or less copying the XML from there.
* Probably other things I'm currently not thinking of

## End

The 2016 feed is at https://teadrivendev.github.io/public/fsadvent2016.rss; I will update it for the few remaining days of the 2016 calendar. So in case you find this useful (and don't use Feedly), you can just use that.

The raw script code from this article can be found in [this gist](https://gist.github.com/TeaDrivenDev/7978ea35093948bb683b14a8d6f3abc9).


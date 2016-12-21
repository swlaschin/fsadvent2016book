


# Advent 2016 #

*All text and code copyright (c) 2016 by Michael Newton. Used with permission.*

*Original post dated 2016-12-16 available at http://blog.mavnn.co.uk/advent-2016/*

**Michael Newton**


Each year I like to make my F# advent post centered around an aspect of the actual Christmas story, so this year I decided to look at the actual text of the Christmas story.

There are a couple of direct historical accounts recorded in the bible, in the Gospels of Mark and Luke. But Jesus's birth is a central point of the overall biblical story, with links to the Old Testiment books written before and referenced in places through the New Testiment.

Sounds like a graph to me, so lets see how far we can take some analysis.

Fortunately, someone has already produced a [text file with a whole bunch of cross references in a nice regular format](https://www.openbible.info/labs/cross-references/). So all we need to get started is a nice parser. We'll also want to pull in some metadata about the structure of the bible as a book in JSON format from the people at [bibles.org](http://bibles.org/pages/api/documentation).

Time to reference some dependencies to do our heavy lifting for us: FParsec for parsing, and FSharp.Data for the JSON type provider.

I'm writing this in the excellent Jupyter F# notebook (and then exporting it as markdown), so I'll use their Paket helpers to grab my dependencies (this should work on the Azure notebooks as well, although I've only tried it locally).

```fsharp
#load "Paket.fsx"

Paket.Package ["FParsec"; "FSharp.Data"]

#r @"packages/FParsec/lib/net40-client/FParsecCS.dll"
#r @"packages/FParsec/lib/net40-client/FParsec.dll"
#r @"packages/FSharp.Data/lib/net40/FSharp.Data.dll"
#r @"packages/FSharp.Data/lib/net40/FSharp.Data.DesignTime.dll"
#r @"packages/Zlib.Portable/lib/portable-net4+sl5+wp8+win8+wpa81+MonoTouch+MonoAndroid/Zlib.Portable.dll"
```

So. This is F#, so the first thing we'll be wanting is our domain model. What do we need to represent our data in the type system?

Well, there is a standardized system for referencing locations in biblical text; we give a book of the bible (the bible is actually a mini-library of sub-books written at different times), the chapter (in theory a thematic block within a book) and a verse (a fairly arbitrary devision of a sentence or two of text). The chapter and verse devisions were not added by the authors, but by monks and scholars well after the fact, but they do give us an accurate way of pointing to a small defined chunk of biblical text between different printings and translations.

It's also frequently useful to refer to a range of verses with in a book.

So we're going to define three main types: `ChapterAndVerse` (what it sounds like), `Location` (book name and `ChapterAndVerse`) and `Range` (book name, start `ChapterAndVerse` and end `ChapterAndVerse`).

We'll use units of measure to make sure we can't swap chapters for verses by mistake, and add some helper methods to give nice string representations of the types and a concise syntax for creating instances of the types.

As an aside: the ordering of the books within the bible is fairly arbitrary, so a range that crosses between two books is meaningless. This is why a range is between to `ChapterAndVerses`, not between two `Locations` - remember folks, make illegal states unrepresentable.

```fsharp
[<Measure>] type Chapter
[<Measure>] type Verse
[<Measure>] type c = Chapter
[<Measure>] type v = Verse

type ChapterAndVerse =
  { Chapter : int<Chapter>
    Verse : int<Verse> }
  static member Make c v =
    { Chapter = c
      Verse = v }
  override x.ToString () =
    sprintf "%d.%d" (x.Chapter / 1<c>) (x.Verse / 1<v>)

let cv =
  ChapterAndVerse.Make

type Location =
  { Book : string
    ChapterAndVerse : ChapterAndVerse }
  static member Make b c v =
    { Book = b
      ChapterAndVerse = { Chapter = c; Verse = v } }
  override x.ToString () =
     sprintf "%s.%O" x.Book x.ChapterAndVerse

let loc =
  Location.Make

type Range =
  { Book : string
    Start : ChapterAndVerse
    End : ChapterAndVerse }
  static member Make b s e =
    { Book = b
      Start = s
      End = e }
  override x.ToString () =
    sprintf "%s.%O-%s.%O" x.Book x.Start x.Book x.End

let r =
  Range.Make
```


We'll also need to be able to test if a location is within a range:

```fsharp
let contains (range : Range) location =
  let lower = { Location.Book = range.Book; ChapterAndVerse = range.Start }
  let upper = { Location.Book = range.Book; ChapterAndVerse = range.End }
  location >= lower && location <= upper
```
  
And get a feel of how this all works.

```fsharp
contains (r "Gen" (cv 1<c> 1<v>) (cv 2<c> 10<v>)) (loc "Gen" 1<c> 2<v>)
```

```fsharp
true
```

Genesis 1:2 (verse 2 of chapter 1 of the book of Genesis) is indeed within the range Genesis 1:1-2:10.

```fsharp
contains (r "Gen" (cv 1<c> 1<v>) (cv 2<c> 10<v>)) (loc "Gen" 3<c> 2<v>)
```

```fsharp
false
```

While Genesis 3:2 is not. Good stuff.

We have a working domain now, let's have a look at the input data.

```fsharp
open System.IO
let input =
  File.ReadAllLines <| Path.Combine(__SOURCE_DIRECTORY__, "cross_references.txt")

input
```


```
[|"From Verse   To Verse    Votes   #www.openbible.info CC-BY 2016-12-05";
  "Gen.1.1  Heb.11.3    67"; "Gen.1.1   1Chr.16.26  11"; "Gen.1.1   Eccl.12.1   4";
  "Gen.1.1  Prov.3.19   19"; "Gen.1.1   Ps.124.8    16"; "Gen.1.1   Isa.65.17   8";
  "Gen.1.1  Ps.104.24   17"; "Gen.1.1   Ps.121.2    14"; "Gen.1.1   Rev.14.7    23";
  "Gen.1.1  Isa.40.26   18"; "Gen.1.1   Rev.3.14    9"; "Gen.1.1    Job.38.4    43";
  "Gen.1.1  Exod.20.11  34"; "Gen.1.1   Isa.37.16   17"; "Gen.1.1   Prov.16.4   14";
  "Gen.1.1  Ps.104.30   14"; "Gen.1.1   Col.1.16-Col.1.17   36"; "Gen.1.1   1John.1.1   14";
  "Gen.1.1  Isa.45.18   53"; "Gen.1.1   2Pet.3.5    26"; "Gen.1.1   Rom.1.19-Rom.1.20   15";
  "Gen.1.1  Isa.44.24   26"; "Gen.1.1   Ps.115.15   21"; "Gen.1.1   Mark.13.19  14";
  "Gen.1.1  Isa.42.5    42"; "Gen.1.1   Ps.134.3    14"; "Gen.1.1   Rev.21.6    3";
  "Gen.1.1  Jer.51.15   21"; "Gen.1.1   Rev.22.13   4"; "Gen.1.1    Ps.33.6 15";
  "Gen.1.1  Isa.51.13   17"; "Gen.1.1   Isa.40.28   17"; "Gen.1.1   John.1.1-John.1.3   56";
  "Gen.1.1  Ps.89.11-Ps.89.12   16"; "Gen.1.1   Ps.90.2 18"; "Gen.1.1   Matt.11.25  2";
  "Gen.1.1  Jer.32.17   21"; "Gen.1.1   Ps.148.4-Ps.148.5   16"; "Gen.1.1   Rev.10.6    18";
  "Gen.1.1  Ps.96.5 13"; "Gen.1.1   Isa.51.16   17"; "Gen.1.1   Jer.10.12   21";
  "Gen.1.1  Ps.102.25   18"; "Gen.1.1   Rom.11.36   14"; "Gen.1.1   Acts.14.15  21";
  "Gen.1.1  Job.26.13   9"; "Gen.1.1    Eph.3.9 14"; "Gen.1.1   Rev.4.11    44";
  "Gen.1.1  Ps.33.9 17"; "Gen.1.1   Neh.9.6 28"; "Gen.1.1   Ps.146.6    17";
  "Gen.1.1  Heb.3.4 15"; "Gen.1.1   Heb.1.2 19"; "Gen.1.1   Heb.1.10    41";
  "Gen.1.1  Ps.136.5    24"; "Gen.1.1   Zech.12.1   16"; "Gen.1.1   Exod.31.18  -11";
  "Gen.1.1  Prov.8.22-Prov.8.30 19"; "Gen.1.1   Acts.17.24  39";
  "Gen.1.1  Acts.4.24   16"; "Gen.1.1   1Cor.8.6    18"; "Gen.1.1   Ps.8.3  17";
  "Gen.1.2  Jer.4.23    17"; "Gen.1.2   Ps.33.6 1"; "Gen.1.2    Job.26.14   0";
  "Gen.1.2  Nah.2.10    -5"; "Gen.1.2   Job.26.7    0"; "Gen.1.2    Isa.40.12-Isa.40.14 -2";
  "Gen.1.2  Isa.45.18   7"; "Gen.1.2    Ps.104.30   9"; "Gen.1.3    1John.2.8   3";
  "Gen.1.3  Job.36.30   1"; "Gen.1.3    Ps.33.9 4"; "Gen.1.3    Eph.5.8 1";
  "Gen.1.3  Isa.60.19   7"; "Gen.1.3    2Cor.4.6    14"; "Gen.1.3   Isa.45.7    7";
  "Gen.1.3  Ps.97.11    4"; "Gen.1.3    John.11.43  -1"; "Gen.1.3   1John.1.5   8";
  "Gen.1.3  John.1.9    2"; "Gen.1.3    Job.38.19   3"; "Gen.1.3    John.3.19   3";
  "Gen.1.3  Ps.33.6 6"; "Gen.1.3    John.1.5    9"; "Gen.1.3    Matt.8.3    -2";
  "Gen.1.3  Ps.148.5    5"; "Gen.1.3    Eph.5.14    2"; "Gen.1.3    Ps.104.2    2";
  "Gen.1.3  Ps.118.27   1"; "Gen.1.3    1Tim.6.16   1"; "Gen.1.4    Gen.1.18    4";
  "Gen.1.4  Gen.1.10    1"; "Gen.1.4    Eccl.11.7   1"; "Gen.1.4    Gen.1.25    1";
  "Gen.1.4  Gen.1.31    1"; "Gen.1.4    Eccl.2.13   2"; "Gen.1.4    Gen.1.12    2";
  "Gen.1.5  Gen.1.23    2"; ...|]
```
  
Looking fairly straight forward here; in fact, after a brief search I realised the format versed here is actually based on a standard called [OSIS](http://www.crosswire.org/osis/), although without all of the unneeded XML bits. Good call.

"Votes" is taken from the original source of these cross references, a context in which it was possible for people to agree or disagree on whether the verses in question are actually linked. For this post I'm just going to ignore the votes, although they'd make an interesting weighting for future investigations.

Time to build a parser to turn this text format into our nice domain types.

```fsharp
open FParsec

let makeString chars =
  chars |> Array.ofList |> System.String

// parse a location
let plocation =
  let dot = pchar '.'
  let notDot = satisfy (fun c -> c <> '.')
  // Take at least one non-dot character.
  // Build a string from the characters taken.
  // This is the book name.
  (many1 notDot |>> makeString)
  // Check the book name is followed by a dot and
  // then throw it away (.>> only keeps the result
  // from the left)
  .>> dot
  // Parse a int; keep both the book and the int.
  // This is the chapter (.>>. keeps left and right
  // results, puts them in a tuple)
  .>>. pint32
  // As above.
  .>> dot
  // Capture the verse int; we now have a tuple on
  // the left so we end up with a tuple containing
  // a tuple and an int
  .>>. pint32
  // Map the awkward tuple structure into our nice
  // domain type ``Location`` if we've found all of
  // the required data
  |>> (fun ((book, chapter), verse) ->
       loc book (chapter * 1<c>) (verse * 1<v>))

// Parse a range
let prange =
  // In the format above, a range is either a single
  // verse (representation same as a location) or as
  // a start and an end location separated by "-".
  // We'll create a parser for the optional second half
  // first...
  let endOfRange =
    pchar '-'
    // >>. throws away the result from the left
    >>. plocation
  // Here we take the start and optional end location
  // and then map them to our domain type.
  plocation .>>. (opt endOfRange)
  >>= (fun (starting, ending) ->
  fun stream ->
    let ending = defaultArg ending starting
    if starting.Book <> ending.Book then
      Reply(Error, expectedString "Both ends of range should be in the same book")
    else
      r starting.Book starting.ChapterAndVerse ending.ChapterAndVerse
      |> Reply)

// Finally, a row from the text file is just a location
// followed by space, followed by a range, then a space
// and the votes. Spaces and votes are ignored by careful
// use of discarding operators.
let row : Parser<_, unit> =
  plocation
  .>> spaces1
  .>>. prange
  .>> spaces1
  .>> pint32
  .>> eof
```
  
A few trials (of correct and incorrect inputs) suggest that our parser is working nicely.

```fsharp
"Gen.1.1" |> run plocation
```

```fsharp
Success: {Book = "Gen";
 ChapterAndVerse = {Chapter = 1;
                    Verse = 1;};}
```

```fsharp
"Gen.1.1-Gen.3.10" |> run prange
```

```fsharp
Success: {Book = "Gen";
 Start = {Chapter = 1;
          Verse = 1;};
 End = {Chapter = 3;
        Verse = 10;};}
```

```fsharp
"Gen.1.1-Heb.11.3" |> run prange
```

```fsharp
Failure:
Error in Ln: 1 Col: 17
Gen.1.1-Heb.11.3
                ^
Note: The error occurred at the end of the input stream.
Expecting: 'Both ends of range should be in the same book'
```

```fsharp
"Gen.1.1 Heb.11.3    67" |> run row
```

```fsharp
Success: ({Book = "Gen";
  ChapterAndVerse = {Chapter = 1;
                     Verse = 1;};}, {Book = "Heb";
                                     Start = {Chapter = 11;
                                              Verse = 3;};
                                     End = {Chapter = 11;
                                            Verse = 3;};})
```

```fsharp
"Gen.1.1.22  Heb.11.3    67" |> run row
```

```fsharp
Failure:
Error in Ln: 1 Col: 8
Gen.1.1.22      Heb.11.3        67
       ^
Expecting: whitespace
```

Now let's run our parser over the input file, and get ourselves a list of cross references.

There's quite a few of them, so we'll only display the first 10.

```fsharp
let crossReferences =
  input
  |> Seq.skip 1 // skip the row titles
  |> Seq.map (run row)
  |> Seq.choose (function
                 | Success (reference, _, _) -> Some reference
                 | _ -> None)

crossReferences
|> Seq.take 10
|> Util.Table
```

| Item1	| Item2 |
| ----	| ---- |
| Gen.1.1	| Heb.11.3-Heb.11.3     |
| Gen.1.1	| 1Chr.16.26-1Chr.16.26 |
| Gen.1.1	| Eccl.12.1-Eccl.12.1   |
| Gen.1.1	| Prov.3.19-Prov.3.19   |
| Gen.1.1	| Ps.124.8-Ps.124.8     |
| Gen.1.1	| Isa.65.17-Isa.65.17   |
| Gen.1.1	| Ps.104.24-Ps.104.24   |
| Gen.1.1	| Ps.121.2-Ps.121.2     |
| Gen.1.1	| Rev.14.7-Rev.14.7     |
| Gen.1.1	| Isa.40.26-Isa.40.26   |


This is all great and everything, but I'd also like to have some way of mapping out where these verses are in the Bible and using the full names of the books. I didn't feel like entering all the data by hand, but fortunately [bibles.org](http://bibles.org/) have done it for me. Time to break out that json type provider…

```fsharp
type BibleInfo = FSharp.Data.JsonProvider<"./books.js">
```

```fsharp
let bibleInfo = BibleInfo.Load("./books.js")
```


And create a map from short to full names.

```fsharp
let bookMap =
  bibleInfo.Response.Books
  |> Seq.map (fun b -> b.Abbr, b.Name)
  |> Map.ofSeq

bookMap
```

```
map
  [("1Chr", "1 Chronicles"); ("1Cor", "1 Corinthians"); ("1Esd", "1 Esdras");
   ("1John", "1 John"); ("1Kgs", "1 Kings"); ("1Macc", "1 Maccabees");
   ("1Pet", "1 Peter"); ("1Sam", "1 Samuel"); ("1Thess", "1 Thessalonians"); ...]
```
   
Now we have all of the actual data we need.

We'll start from the historical accounts of Jesus's actual birth…

```fsharp
let story = [
    // Matthew's account
    r "Matt" (cv 1<c> 18<v>) (cv 2<c> 18<v>)

    // Luke's account part 1
    r "Luke" (cv 1<c> 26<v>) (cv 1<c> 56<v>)

    // Luke's account part 2
    r "Luke" (cv 2<c> 1<v>) (cv 2<c> 21<v>)
  ]
```
  
…and then find all of the cross references which come from a verse in these ranges…

```fsharp
let findCrossReferencesOfRange references range =
  references
  |> Seq.filter (fun reference -> contains range <| fst reference)
```

```fsharp
let allCrossReferences references ranges =
  ranges
  |> Seq.fold (fun found range ->
     Seq.concat [findCrossReferencesOfRange references range;found]) Seq.empty
```
	
…giving us a sequence of every range cross referenced from the Christmas story.

```fsharp
let christmasRefs =
  allCrossReferences crossReferences story
```

```fsharp
christmasRefs |> Seq.length
```

```fsharp
1042
```

```fsharp
christmasRefs
|> Seq.take 10
|> Util.Table
```


| Item1	| Item2
| Luke.2.1	| Acts.11.28-Acts.11.28 |
| Luke.2.1	| Matt.24.14-Matt.24.14 |
| Luke.2.1	| Acts.25.21-Acts.25.21 |
| Luke.2.1	| Acts.25.11-Acts.25.11 |
| Luke.2.1	| Rom.1.8-Rom.1.8       |
| Luke.2.1	| Luke.3.1-Luke.3.1     |
| Luke.2.1	| Phil.4.22-Phil.4.22   |
| Luke.2.1	| Mark.16.15-Mark.16.15 |
| Luke.2.1	| Matt.22.17-Matt.22.17 |
| Luke.2.1	| Mark.14.9-Mark.14.9   |

We're getting closer. Before we dive into the graph, we'll have a quick poke around the data.

Like, which are the top 10 books most commonly cross referenced to?

```fsharp
// count per book, sort by descending
let byBook =
  christmasRefs
  |> Seq.map (fun r -> bookMap.[(snd r).Book])
  |> Seq.countBy id
  |> Seq.sortByDescending snd

byBook |> Seq.take 10 |> Util.Table
```


| Item1	| Item2 | 
| Luke	    | 129  |
| Matthew	| 115  |
| Psalm     |  101 |
| Isaiah    | 74   |
| Acts      | 68   |
| John      | 66   |
| Genesis   | 52   |
| Revelation | 33  |
| Exodus     | 27  |
| Jeremiah   | 26  |


Getting closer to the data we want to create an edge and node graph from, let's look at the links between books.

```fsharp
// Count how many times a book is referenced from each of our
// source books
let fromBookToBook =
  christmasRefs
  |> Seq.map (fun (from, to') -> bookMap.[from.Book], bookMap.[to'.Book])
  |> Seq.countBy id
  |> Seq.sortByDescending snd

fromBookToBook
|> Seq.take 10
|> Util.Table
```


| Item1            | Item2 |
| (Luke, Luke)       | 94 |
| (Luke, Psalm) 	 | 72 |
| (Matthew, Matthew) | 63 |
| (Luke, Matthew)    | 52 |
| (Luke, Isaiah)     | 52 |
| (Luke, Acts)       | 37 |
| (Luke, John)       | 35 |
| (Matthew, Luke)    | 35 |
| (Matthew, Acts)    | 31 |
| (Matthew, John)    | 31 |

And now we've arrived. I've not found a nice simple way of generating graph images in .net, so we'll turn to the well used d3 javascript library to help us out, given we're running in a web page.

We need to also to have a way of turning our data into a valid javascript representation. We'll need to feed d3 an array of nodes (just the names of the books of the Bible). Then we'll have an array of "links", which we'll also give a "strength" field to represent the number of cross references. The links need to use the index of the source and target in the node list.

```fsharp
let names =
  bibleInfo.Response.Books
  |> Seq.map (fun b -> b.Name)

// Simple string concatination will do the job here,
// for more complex data we could use an actual JSON
// serialization library
let nodes =
  names
  |> Seq.map (fun name -> sprintf "{ id: %A, x: 0, y: 0 }" name)
  |> String.concat ",\n"
  |> sprintf "[%s]"

let links =
  fromBookToBook
  |> Seq.map (fun ((source, target), strength) ->
    let sourceI = Seq.findIndex ((=) source) names
    let targetI = Seq.findIndex ((=) target) names
    sprintf "{ source: %d, target: %d, strength: %d }" sourceI targetI strength)
  |> String.concat ",\n"
  |> sprintf "[%s]"
```
  
Now we need some javascript and html to get this all up and running. Let's make sure we have d3 loaded on the page.

```fsharp
"""<script src='http://d3js.org/d3.v3.min.js'></script>""" |> Util.Html |> Display
```

And now we can just enter our raw javascript to create the graph, with the node and link data from above. Hopefully most of this will make some sense on a read through but the basic flow is:

1. Inject an svg element into our page
1. Create a force layout with some standard properties (gravity to keep everything near the middle, charge to keep nodes from overlapping)
1. Make the desired distance between linked nodes shorter the "stronger" the link is.
1. Add lines for links, and circles and text for nodes.
1. Fire a call back to reposition the links and nodes as the force simualtion runs

If you've read down here from the beginning, the graph has probably settled into a steady state, but feel free to reload the page and watch the nodes bounce around.

```fsharp
sprintf """
<style>
.node {
    fill: #5cc;
    stroke: #2aa;
    stroke-width: 2px;
}

.link {
    stroke: #777;
    stroke-width: 2px;
}
</style>


<div id="viz"></div>

<script>
var width = 800;
var height = 800;

var nodeData = %s;

var linkData = %s;

var force = null,
    nodes = null,
    nodeTitles = null,
    links = null;

var svg = d3.select('#viz').append('svg')
    .attr('width', width)
    .attr('height', height);
    
var initForce = function() {
    svg.selectAll('*').remove();

    force = d3.layout.force()
        .size([width, height])
        .nodes(nodeData)
        .links(linkData)
        .gravity(0.5)
        .charge([-1000]);
        
    force.linkDistance(function (link) { return 400 / link.strength });

    links = svg.selectAll('.link')
        .data(linkData)
        .enter().append('line')
        .attr('class', 'link');
    
    nodes = svg.selectAll('.node')
        .data(nodeData)
        .enter().append('circle')
        .attr('class', 'node')
        .attr('r', width / 50);
        
    nodeTitles = svg.selectAll('text')
        .data(nodeData)
        .enter().append('text')
        .attr('text-anchor', 'middle')
        .text(function (d) { return d.id })
        .attr('font-family', 'sans')
        .attr('font-size', 16)
        .attr('fill', 'black');
    
    force.on('tick', stepForce);
};

var stepForce = function() {

    nodes.attr('cx', function(d) { return d.x; })
        .attr('cy', function(d) { return d.y; });

    nodeTitles.attr('x', function(d) { return d.x; })
        .attr('y', function(d) { return d.y; });

    links.attr('x1', function(d) { return d.source.x; })
        .attr('y1', function(d) { return d.source.y; })
        .attr('x2', function(d) { return d.target.x; })
        .attr('y2', function(d) { return d.target.y; });
};

initForce();
force.start();
</script>
""" nodes links
|> Util.Html
```

![](d3_snapshot.png)

*NOTE: This is a snapshot -- to see the live force layout graph in actions, visit the corresponding page for this post: http://blog.mavnn.co.uk/advent-2016/*

And there it is. A nice force layout graph based on our F# data, displaying the properties you would expect. Matthew and Luke as the "source" nodes have settled somewhere near the centre, with books commonly referenced from both squeezed in between. An outer ring of books referenced infrequently or from only one of the other form the next ring, and then around the edges we have the books not referenced at all during the Christmas story.

I hope you enjoyed this magical mystery tour of Jupyter, d3 and the Christmas story!




# The magic of type providers #

*All text and code copyright (c) 2016 by Roman Nevolin. Used with permission.*

*Original post dated 2016-12-24 available at https://medium.com/@nevoroman/the-magic-of-type-providers-7f6825acd54#.nj958l2cq*

**By Roman Nevolin**

Every programmer sometimes processing many different formats of data -- it can be some JSON api response, SQL data or XML configuration. And it’s always boring, routine and problematic process. Is there at least one programmer who didn’t make typo in names or types of JSON fields?

Let’s do something with StackOverflow API response -- it returns JSON For example, print titles of last questions. In C# it can look like this:

```csharp
public void PrintSOQuestions() {
    const string questionsUrl = "https://api.stackexchange.com/2.2/questions?site=stackoverflow";
    var handler = new HttpClientHandler {
        AutomaticDecompression = DecompressionMethods.GZip | DecompressionMethods.Deflate
    };
    using ( var http = new HttpClient( handler ) ) {
        var json = http.GetStringAsync( questionsUrl ).Result;
        var questions = JsonConvert.DeserializeObject<Result<Question>>( json ).Items;
        foreach ( var question in questions ) {
            Console.WriteLine(question.Title);
        }
    }
}

public class Result<T> {
    public T[] Items { get; set; }
    public bool HasMore { get; set; }
    public int QuotaMax { get; set; }
    public int QuotaRemaining { get; set; }
}

public class Question {
    public string[] Tags { get; set; }
    public User Owner { get; set; }
    public bool IsAnswered { get; set; }
    public int ViewCount { get; set; }
    public int AnswerCount { get; set; }
    public DateTime ProtectedDate { get; set; }
    public long AcceptedAnswerId { get; set; }
    public int Score { get; set; }
    public DateTime LastActivityDate { get; set; }
    public DateTime CreationDate { get; set; }
    public DateTime LastEditDate { get; set; }
    public long QuestionId { get; set; }
    public string Link { get; set; }
    public string Title { get; set; }
}

public class User {
    public int Reputation { get; set; }
    public long UserId { get; set; }
    public string UserType { get; set; }
    public int AcceptRate { get; set; }
    public string ProfileImage { get; set; }
    public string DisplayName { get; set; }
    public string Link { get; set; }
}
```

There’s not really much logic, but a lot of boiler-plate code for data models. Of course, we can use some language with dynamic type checking -- Python, for example. And this code will be much shorter:

```
questionsUrl = "https://api.stackexchange.com/2.2/questions?site=stackoverflow"
compressedJson = urllib.request.urlopen(questionsUrl).read()
decompressedJson = gzip.decompress(compressedJson)

questions = json.loads(decompressedJson.decode())['items']
for question in questions: print(question['title'])
```

But now we have all problems of dynamic type system -- for example, autocompletion can’t help us here. And, of course, it’s much easier to do some typo.

So we can’t say, that it really better. But maybe there is some way to combine advantages of both variants? Something like this:

```fsharp
let [<Literal>] questionsUrl = """https://api.stackexchange.com/2.2/questions?site=stackoverflow"""

type Question = JsonProvider<questionsUrl>
let questions = Question.Load questionsUrl
questions.Items |> Array.iter (fun x -> printfn "%s" x.Title) 
```

Here we aren’t writing any data models, but have all advantages of static type checking -- as you can see at screenshot below, IntelliSense give us full autocomplete, with correct names and types.

![](1.png)

You see? We haven’t implemented any data models, but… it works! How?

All this magic possible because of F # language mechanism, named “Type providers”. Every type providers -- it’s some library, that provides functionality to work with certain type. By the way, we can easily replace one provider by the other. For example, if SO API will return us XML response, code will be almost unchanged:

```fsharp
let [<Literal>] questionsUrl = """https://api.stackexchange.com/2.2/questions?site=stackoverflow"""

type Question = XmlProvider<questionsUrl>
let questions = Question.Load questionsUrl
questions.Rows |> Array.iter (fun x -> printfn "%s" x.Title) 
```

How does this magic work? Let’s take a look at the use of JSON provider. When we creating it, we pass the example JSON URL. JsonProvider pulls it, determines its schema and create corresponding F# types. After that F# Compiler catch created types and we can use all the advantages of strong type checking, like autocomplete. Now it’s possible to map any JSON with appropriate schema to created types.

It can also decide some other problems. For example, if API will be changed, your code will stop stop working anyway. But if you are using type providers, you’ll know about it on compilation stage -- because type provider will load new data scheme.

So, type providers are awesome. I’ll show you, how much it can be awesome with some other examples.

## Not only for reading

It’s obvious that sometimes we need more than reading data. A good example is interaction with SQL database -- we need some other functions here, like updating or deleting records. It would not be too good if type providers can’t do simple CRUD operations… but, of course, type providers can.

SQL Provider can easily do all CRUD operations -- it is easy to compare with ORM. To show how it works let’s solve some simple problem. For example fill “IsHoliday” field of Sales table, according to “Date” field using SQL Provider

```fsharp
let [<Literal>] connectionString = "Data Source=(local);Initial Catalog=Stats;Integrated Security=True;"
type StatsDB = SqlDataProvider< 
                  ConnectionString = connectionString,
                  DatabaseVendor = Common.DatabaseProviderTypes.MSSQLSERVER>

let context = StatsDB.GetDataContext()

context.Dbo.Sales |> Seq.iter(fun sale ->
    sale.IsWeekend <- sale.date.DayOfWeek = DayOfWeek.Saturday 
                      || sale.date.DayOfWeek = DayOfWeek.Sunday
    context.SubmitUpdates()
)
```

It really looks like ORM, isn’t it? But we don’t create any data models.

## Not only for simple data types

It isn’t really hard to parse something like JSON, but what about other data sources, like HTML? Anyone who has ever parsed HTML, know -- this is unusually dull and dreary task. It’s so easy to make a stupid mistake in selector!

Fortunately, we can use type providers to avoid HTML parsing problems. There is HTML provider which allows you to use all advantages of strong type checking when you working with HTML. Let’s use it for some simple real-life task. Recently I had to get a list of heads of government of members of the United Nations. Let’s try to parse it from Wikipedia.

```fsharp
let [<Literal>] wikipageUrl = """https://en.wikipedia.org/wiki/List_of_current_heads_of_state_and_government"""
type HeadsOfStatePage = HtmlProvider<wikipageUrl>

HeadsOfStatePage.Load(wikipageUrl)
    .Tables
    .``Member and observer states of the United Nations``
    .Rows 
    |> Array.map(fun x -> x.State, x.``Head of government``)
```    

I didn’t even look at HTML-code of Wiki page, HTML provider gave me all the necessary information. And the code has turned out much more readable than if we had used for this purpose more common tools like HTML Agility Pack.

## Not only for data

In fact, type providers functionality are not limited by data manipulations. There are other, much more interesting cases.

For instance, cross-language interoperability. As you know, it can be challenging task with a lot of pitfalls. But we can do it much easier through type providers.

R provider shows this perfectly. As you can guess, it makes possible to use R capabilities from the F#. Let’s take a look at usage example from R Provider documentation

```fsharp
type Stocks = CsvProvider<"http://ichart.yahoo.com/table.csv?s=MFST"> 

/// Returns prices of a given stock for a specified number 
/// of days (starting from the most recent)
let getStockPrices stock count =
  let url = "http://ichart.finance.yahoo.com/table.csv?s="
  [| for r in Stocks.Load(url + stock).Take(count).Rows -> float r.Open |] 
  |> Array.rev

// Build a list of tickers and get diff of logs of prices for each one
let tickers = 
  [ "MSFT"; "AAPL"; "X"; "VXX"; "GLD" ]
let data =
  [ for t in tickers -> 
      t, getStockPrices t 255 |> R.log |> R.diff ]

// Create an R data frame with the data and call 'R.pairs'
let df = R.data_frame(namedParams data)
R.pairs(df)
```

As you can see, this type provider allows you to use functions of another language as some of the NuGet libraries. By the way, this example shows you how several type providers can be used together.

## Conclusion

So, type providers can help you to write very efficient and reliable code. Of course, I haven’t shown all of the possible type provider usages -- they are really limitless. There is type providers to interact with JavaScript, Swagger and much more. And nothing prevents you from creating you own type provider -- excellent recommendations on this subject can be found in [Sergey Tihon’s blog.](https://sergeytihon.wordpress.com/2016/07/11/f-type-providers-development-tips-not-tricks/)


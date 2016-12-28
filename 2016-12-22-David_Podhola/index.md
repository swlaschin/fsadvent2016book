



# Hello fable-import-sharepoint #

*All text and code copyright (c) 2016 by David Podhola. Used with permission.*

*Original post dated 2016-12-22 available at http://www.naseukoly.cz/blog/2016/12-20-hello-fable-import-sharepoint/index.html*

**By David Podhola**


This post is about 3 month long journey to [fable-import-sharepoint](https://github.com/hsharpsoftware/fable-import-sharepoint). It is part of F# Advent 2016. Includes also a lot of information from my past.

## TL;DR

Using [Fable](http://fable.io/) and [fable-import-sharepoint](https://github.com/hsharpsoftware/fable-import-sharepoint) you can easily create complex multisite SharePoint applications that make big changes to the user interface, start workflows from the browser etc.

If you want to ask questions, report issues or contribute, please feel free to at the [fable-import-sharepoint GitHub repository](https://github.com/hsharpsoftware/fable-import-sharepoint).

## Preface

Two months ago I became a part of a project team assigned a very interesting task - to rewrite a highly customized DMS on a top of Microsoft SharePoint. I was not for the first time I was writing for SharePoint - in fact, it was my fourth project this year, but this time it was a little bit special. The projects we wrote during the summer and autumn were [provider-hosted SharePoint Add-ins](https://msdn.microsoft.com/en-us/library/office/fp142381.aspx). We wrote them as server applications running on the top of [Suave hosted in IIS](https://blog.vbfox.net/2016/03/31/Suave-in-IIS-hello-world.html) with the front end using [React](https://facebook.github.io/react/) and [Redux](http://redux.js.org/). These are the technologies we have been using for years and [were proven to work for us and help us to write code faster](http://fsharp.org/testimonials/#synctoday-1).

## SharePoint Designer Only

The new project was completely different. From all the SharePoint integration and customization options, only those accessible using SharePoint Designer were allowed. At the same time the user requirements were quite high which was understandable taking into the account they were used to the application created directly for their needs. Most of them were related to the user experience in terms of:

* information they see on the screen (add or remove)
* actions (links, buttons, menu items) accessible on the particular screen (add or remove)
* specialized validation (file name including wild cards against a specific list)
* multiple file upload
* cascade workflows with App-Steps
* very strict user permissions (e.g. upload only, cannot read the file later)

It was clear soon that not only heavy UI changes will need to be done, but also iterating for list items, starting workflows and so on.

## JavaScript Legacy

There are different ways how to modify SharePoint pages. I did not want to modify HTML directly mainly because the application is in fact running on multiple sites so it would be extremely difficult to sync the changes. The other ways was to deploy JavaScript to Site Assets, include it in the MasterPage and let it add or hide the elements that we needed.

Now let me get back in the history for a while. I first met JavaScript in about 2001 when we were developing large Dealer Management System for BMW importer and dealers. At that time we switched from writing ASP applications backed with Delphi COM+ components to the very first betas of Microsoft .NET. There were no ASP.NET yet and we wanted to write an intranet/extranet application so we have to invent an application architecture by ourselves.

We were very much influenced by that time latest news from Microsoft like [SQL XML features](https://msdn.microsoft.com/en-us/library/ms950789.aspx) so we decided most of the logic would be done by the SQL server generating XML that was later transformed by static XSLT files into the final XHTML.

Everything was very fine except for the users demanding some advanced features on the client side like the suggestions as they typed, validations before form submit etc. And that was when we started to write JavaScript.

Of course it was terrible. There were no jQuery at that time. No advanced frameworks like Angular or React. Niente. Just the plain JavaScript.

For the developers with the Delphi, Visual Basic and later C# background with static types and the compile step checking for typos etc. it was simply a nightmare. Endless nights when we were trying to hunt bugs strange bugs in Internet Explorer. All the funny stories about comparing apples to oranges and getting [the most difficult-to-understand results you can imagine](http://charlieharvey.org.uk/page/javascript_the_weird_parts).

So it is not difficult to imagine for the next 10 years if a feature was requested by a project/product manager the answer was sometimes "It would need to be done in JavaScript, so ..." supported by nodding and affirmative grunts by the survivors of the previous project.

## Modern JavaScript

About ten years later we were writing a ASP.NET MVC application for call centre agents build on the top of Microsoft Dynamics CRM. With the help of jQuery the code was much more readable now, but we still have to fight a lot of bugs in the runtime (and even production as part of the application was dynamically generated!). Not nice.

Heard about something called Angular at that time, but the server side generated HTML approach was still too strong.

Fast forward to mid-2015. We are writing an application for managing the front office clerks and their shifts. We decided to use ASP.NET Core and Angular. Had a lot of fun and although I did mostly the back end, but it looked like Angular is very usable and ready to replace the ASP.NET MVC.

When 2016 kicked in, I am helping my friends to add new Angular front end to their ASP.NET WebPages application from late 2000s. It is not my first Angular application, but for the first time (and definitely not for the last) we were adding Angular SPA to server generated application already many years in production.

## F# and FunScript

I have described my path to programming in F# [several times](https://github.com/fsharping/Docs/tree/master/004-FSharp%20v%20praxi). For the past years me and my colleagues have been using F# for:

* writing simple console application
* writing Outlook AddIns
* writing ASP.NET web servers

When we was asked by a customer if we were able to write [Office Add-ins](https://dev.office.com/docs/add-ins/overview/office-add-ins), I was not very happy first imagining we will have to write another JavaScript-only application. Then to my surprise, I found [FunScript](http://funscript.info/). I hoped to use it, I studied to source code and even tried to contribute, but it seems FunScript was already abandoned at that time.

## Fable to the rescue

Then in February 2016, I found about [Fable](http://fable.io/). You can imagine how happy I was. A new world emerged. First small module we wrote was just a helper function working making complex tree structures flat. The results were amazing: few hours writing F# code and the tests. Then transpilling into JavaScript and another few hours tweaking the interface to be really usable from regular JavaScript.

I was already aware [Ionide](http://ionide.io/) is written in F# and uses Fable, but I needed to meet project where I would really use it.

## fable-import-sharepoint

And then the JavaScript-only SharePoint appeared. I immediately decided we would need to have some support. There were three basic areas, where Fable would help:

* using [Visual Studio Code with Ionide](https://docs.microsoft.com/en-us/dotnet/articles/fsharp/tutorials/getting-started/getting-started-vscode) and the autocomplete not to have to remember [all the SharePoint functions by heart](https://msdn.microsoft.com/en-us/library/office/jj163201.aspx)
* creating the Domain Specific Language for the general SharePoint frontend development
* creating specific types and functions related to the project domain (complex DMS with specific workflows and strict permission system)

The first two from the list are shared in the [fable-import-sharepoint GitHub repository](https://github.com/hsharpsoftware/fable-import-sharepoint).

Let me explain, how I created them and what they can be used to.


### Fable.Import.Sharepoint*.fs

These are the files that help [the autocomplete (IntelliSense) provided by Ionide in Visual Studio Core](https://www.youtube.com/watch?v=RGiwiAoIn6w). They were created by running slightly modified `ts2fable` on files from [DefinitelyTyped/SharePoint](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/4869992bc079b88280b9ff91213528904109e8ae/sharepoint/index.d.ts) and then working with the files by hand till the results were what I wanted them to be.

Looks boring? Was not at all! `ts2fable` creates most of the content you need and also in this task creativity is needed.

The interface is far from perfect now, but was usable not only in the initial project, but also in all of the following.

You can use it to write the classic SharePoint snippets like

```fsharp
let clientContext = SP.ClientContext.get_current()
let web = clientContext.get_web()
clientContext.load(web)
```

### Browser.Support.fs

We created `Browser.Support.fs` to make it easier to write jQuery calls in the functional way.

Examples:

* `Literals.Global.Ribbon.ApproveReject |> el |> hide`
* `Literals.Global.Ribbon.S4RibbonRow |> el |> find ( Literals.Global.Ribbon.MsCuiCtlMedium ) |> last |> hide`
* `Literals.AccessRequestRPR.RequestType.ID |> el |> change ( fun () -> showAndHide() ) |> ignore`

Note: this is not related to SharePoint development.

### HSharp.fs

The requirements to change SharePoint user interface on multiple sites led to creating few small support functions in `HSharp.fs`. If you implement `IApplication` interface, you can then add

``` 
<SharePoint:ScriptLink runat="server" ID="NUCZScriptLink1" Language="javascript" Name="~site/SiteAssets/Scripts/MicrosoftAjax.js"/>
<SharePoint:ScriptLink runat="server" ID="NUCZScriptLink2" Language="javascript" Name="~site/SiteAssets/Scripts/jquery-1.7.2.min.js"/>
<SharePoint:ScriptLink runat="server" ID="NUCZScriptLink3" Language="javascript" Name="~site/SiteAssets/Scripts/core.min.js"/>
<SharePoint:ScriptLink runat="server" ID="NUCZScriptLink7" Language="javascript" Name="~site/SiteAssets/Scripts/jquery.SPServices-2014.02.js"/>
<SharePoint:ScriptLink runat="server" ID="NUCZScriptLink4" Language="javascript" Name="~site/_layouts/15/SP.Runtime.js"/>
<SharePoint:ScriptLink runat="server" ID="NUCZScriptLink5" Language="javascript" Name="~site/_layouts/15/SP.js"/>
<SharePoint:ScriptLink runat="server" ID="NUCZScriptLink8" Language="javascript" Name="~site/_layouts/15/SP.WorkflowServices.js"/>
<SharePoint:ScriptLink runat="server" ID="NUCZScriptLink9" Language="javascript" Name="~site/_layouts/15/SP.UI.Dialog.js"/>
<SharePoint:ScriptLink runat="server" ID="NUCZScriptLink10" Language="javascript" Name="~site/SiteAssets/Scripts/dropzone.js"/>
<SharePoint:ScriptLink runat="server" ID="NUCZScriptLink11" Language="javascript" Name="~site/SiteAssets/Scripts/rsvp-latest.js"/>
        <script type="text/javascript">
    // Load the required SharePoint libraries.
    $(document).ready(function () {
        ExecuteOrDelayUntilScriptLoaded(execOperation,"sp.js");
    });

    function execOperation() {
        HSharp.start();
    }
</script>   
<SharePoint:ScriptLink runat="server" ID="NUCZScriptLink6" Language="javascript" Name="~site/SiteAssets/Scripts/bundle.js"/>
<SharePoint:CssRegistration Name="<% $SPUrl:~site/SiteAssets/Scripts/dropzone.css %>" runat="server" />
```

to your MasterPage.

Function `render` is called once time after the page is rendered when all the page is loaded onto the browser. It can be used to hide or show page UI elements, start workflows etc.

Function `scheduled` is scheduled to be called once per a second; can be used to e.g. dynamically change the user interface of the ribbon etc.

## Final thoughts

F# has once again proven to be a great language for custom development in JavaScript. But if you are using F# for Windows development or server side development, consider adding Fable to your stack and generate JavaScript from source code in the language you know and love.


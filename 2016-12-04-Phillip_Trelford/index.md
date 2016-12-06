# X-Platform Development With Xamarin.Forms And F# #

*All text and code copyright (c) 2016 by Phillip Trelford. Used with permission.*

*Original post dated 2016-12-04 available at http://www.trelford.com/blog/post/XamarinForms.aspx*

**By Phillip Trelford**

Last month on November 16 – 17, [IDTechEx](http://www.idtechex.com/) held their largest annual [conference and tradeshow](http://www.idtechex.com/usa) on
emerging technologies in California with over 200 speakers and close to 4,000 attendees.

![](exhibition.jpg)

The show covers a wide variety of topics including electric vehicles, energy harvesting, energy storage, IoT, graphene, printed electronics, robotics, sensors,
AR, VR and wearables. Coming from a software background, I couldn’t help but marvel at the size of the event and the level of genuine innovation and entrepreneurship
on show. If you’re based in Europe I’d highly recommend attending the Berlin event in [May 2017](http://www.idtechex.com/europe).

IDTechEx wanted an app where users can easily browse information, including [research](http://www.idtechex.com/research/),
[journals](http://www.idtechex.com/research/web-journals.asp) and upcoming [webinars](http://www.idtechex.com/research/webinars.asp),
when they’re out and about, even in a WiFi dead spot. The app needed to be released before the show in California so that at a minimum users could download and browse the conference agenda, list of exhibitors and easily find the venue on a map. Xamarin.Forms and F# were successfully used to build the app which was published in the iTunes and Google Play stores a week before the conference, and well received by attendees:

* [iOS version](http://www.idtechex.com/ios)
* [Android version](http://www.idtechex.com/android)

This post will talk about some of the development process and some of the design decisions for this app.

## Xamarin.Forms

IDTechEx wanted a native business app that targeted the vast majority of mobile users which concretely meant iOS and Android support.
[Xamarin.Forms](https://www.xamarin.com/forms) was chosen as it allows business style apps to be built for both iOS and Android where most of the code can be shared. It is hoped that over the life of the product that the use of Xamarin.Forms should help reduce maintenance costs and improve time-to-market.

Check out Xamarin’s pre-built examples for sample code and to see what can be done: https://www.xamarin.com/prebuilt

## Why F# #

Xamarin’s ecosystem, including Xamarin.Forms, supports both the C# and F# programming languages. F# is a pragmatic, low ceremony, code-oriented programming language, and the preferred language choice for IDTechEx and it’s development staff.

If you’re new to F# and interested in learning more, I’d recommend checking out these resources:

* [F# Foundation](http://fsharp.org/)
* [F# for Fun and Profit](https://fsharpforfunandprofit.com/)
* [F# Koans](https://github.com/ChrisMarinos/FSharpKoans)

## Creating an F# cross platform Xamarin.Forms solution

To get started I followed Charles Petzold’s detailed article on [Writing Xamarin.Forms Apps with F#](http://www.charlespetzold.com/blog/2015/10/Writing-Xamarin-Forms-Apps-in-FSharp.html),
where the steps for creating a cross platform solution can be briefly summarised as:

* Make sure you’re on the latest and greatest stable version of Xamarin (and Visual Studio if you’re on Windows)
* Create a C#/.Net >> Cross Platform >> Blank App (Xamarin.Form Portable)
* Replace the C# Portable library with an F# one

## Development environments

During development I used 2 environments off of the same code base:

* Visual Studio 2015 and Windows for testing and deploying the Android version
* Xamarin Studio and Mac for testing and deploying the iOS version

Switching between Xamarin Studio and Visual Studio on the respective operating systems was seamless.

## Build one to throw away

Initially I built a couple of very simple prototype apps using only a a subset of the data, one using XAML and data binding and another using just code. After some playing I came down on the side of the code only approach.

Xamarin.Forms is quite close to WPF, Silverlight and Windows Store application development. I built a working skeleton/prototype as an F# script using WPF and the F# REPL which gives incredibly fast feedback during development.

Once the flow and views of the application were better defined, and I’d received early feedback from users, I threw away the prototype and rebuilt the app fully in Xamarin.Forms.

Note: the [plan to throw one away](http://wiki.c2.com/?PlanToThrowOneAway) referenced in this section heading refers
to Fred Brooks Jr’s suggestion in his seminal book [The Mythical Man Month](https://en.wikipedia.org/wiki/The_Mythical_Man-Month).

## Forms DSL

During development in both WPF and Xamarin.Forms I built up a minimal [domain-specific language](https://en.wikipedia.org/wiki/Domain-specific_language) to simplify form building, for example:

```fsharp
    let label text = Label(Text=text)
    let bold (label:Label) = label.FontAttributes <- FontAttributes.Bold; label
    let italic (label:Label) = label.FontAttributes <- FontAttributes.Italic; label
    let namedSize name (label:Label) = label.FontSize <- Device.GetNamedSize(name, typeof<Label>); label
    let micro label = label |> namedSize NamedSize.Micro
    let small label = label |> namedSize NamedSize.Small
    let medium label = label |> namedSize NamedSize.Medium
    let large label = label |> namedSize NamedSize.Large
```
 
With this the exhibitor page could be written as:

```fsharp
    let exhibitorPage (exhibitor:json) =
        let program = exhibitor?memberOf?programName.AsString()
        let layout =
            StackLayoutEx(VerticalOptions = LayoutOptions.FillAndExpand, Padding=Thickness 10.0)
            + (cachedImage (exhibitor?logo.AsString(), 130.0, 78.0))
            + (exhibitor?name.AsString() |> label |> bold |> medium)
            + (program |> label |> color (programColor program))
            + (exhibitor?location?name.AsString() |> label)
            + ("Company Profile" |> label |> bold)
            + (exhibitor?description.AsString() |> label)
        let url = exhibitor?url.AsString()
        if not <| System.String.IsNullOrEmpty(url)
        then
            let action () = Device.OpenUri(System.Uri(url))
            Button(Text="Website", Command=Command(action)) |> layout.Children.Add
        ContentPage(Title="Exhibitor", Content=layout)
```
 
Below is a screen shot from the Android App of a exhibitor page generated by the code above:
 
![](image.png)

## Lists

For lists of items, for example exhibitors or presentations, I used Xamarin.Form’s built-in [ListView](https://developer.xamarin.com/guides/xamarin-forms/user-interface/listview/) control
and the [ImageCell](https://developer.xamarin.com/guides/xamarin-forms/user-interface/listview/customizing-cell-appearance/#ImageCell)
and [TextCell](https://developer.xamarin.com/guides/xamarin-forms/user-interface/listview/customizing-cell-appearance/#TextCell), along with a custom cell providing more subheadings.

The screen shot below is from the iOS version and shows a list of presentations using a custom cell:

![](presentations.jpg)
 
## Other Libraries

For image loading I used the excellent [FFImageLoading](https://github.com/luberda-molinet/FFImageLoading) which can cache image files on the user’s device.

To show a map for the venue I used Xamarin.Forms.Maps control:

![](map.jpg)

## Summary

Using Xamarin.Forms and F# allowed be to successfully build and publish a cross platform mobile app targeting iOS and Android in a short time frame. F# allowed fast prototyping and quick feedback while building the app. Xamarin.Forms worked well and meant that almost all of the code was shared between the platforms. I’d whole heartedly recommend this approach for similar business applications and we plan to extend the app heavily in future releases.


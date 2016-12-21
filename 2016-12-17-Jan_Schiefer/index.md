


# Accessing native C libraries on Linux with F# and .Net Core #

*All text and code copyright (c) 2016 by Jan Schiefer. Used with permission.*

*Original post dated 2016-12-17 available at http://hamstr.net/f%23/2016/12/16/Accessing-Linux-C-libraries-with-F-and-.Net-Core/*

**By Jan Schiefer**

**Update 18 Dec 2016**: Added link to Andrew Arnott's [pinvoke project](https://github.com/AArnott/pinvoke) that collects P/Invoke definitions (thanks @ChetHusk for the pointer!), and what seems to be an "official" [P/Invoke library for RTL-SDR](https://github.com/librtlsdr/SharpRTL) in C#.

There are many interesting native libraries available on the Linux platform, which are usually accessed directly from C programs. There will likely be more of these in the future; for example, libraries to program custom hardware accelerators, for which no operating system support is available. C is still a perfectly fine programming language for lower-level work, but this is 2016, and we can do better. In this post I would like to introduce a few of the benefits that you gain when you use a modern "multi-paradigm" language like F# to access native libraries. This allows you to write code in interesting ways that you may not have seen before. I am assuming a working knowledge of C and Linux, but not much else.

"Waitwhat, isn't F# some Microsoft thing?" I hear you say. Yes, it came out of Microsoft Research several years ago. In the meantime, it has grown up into a mature [OSS language](https://github.com/fsharp/fsharp) with an [Apache license](https://github.com/fsharp/fsharp/blob/master/LICENSE) and a small-ish, but enthusiastic, friendly and helpful OSS community. F# [compiles into Javascript](https://fable-compiler.github.io/) or the Microsoft .Net platform in one of its many incarnations. I will be focusing on the (somewhat adolescent) [.Net Core](http://dot.net/core), as it has a lot of promise and is available today on Linux, macOS and Windows.

One of the things .Net does really well is integrate with the underlying native platform, using a declarative mechanism called P/Invoke. So even if you don't like Microsoft and the .Net platform (and believe me, I used to be in that camp), please hold your nose and read on. Or even better, install *the LTS version of .Net Core* for your platform, open your shell, and cast the magic spell:

```
dotnet new --lang fsharp
vi Program.fs
```

Wait - did I just say "vi"? Sorry, old habits die hard. I should probably mention that F# *greatly* benefits from a language-aware editor, which can offer assistance as you work with your code. As F# is a strongly typed language, the editor can offer quite a lot of help in identifying incorrect constructs before you even compile. There are F# plugins for [all sorts of editors](http://fsharp.org/guides/mac-linux-cross-platform/#editing) ([even Emacs, if you must](https://github.com/fsharp/emacs-fsharp-mode))). If you don't know where to start, try the excellent [Visual Studio Code](https://code.visualstudio.com/) with the awesome [Ionide-fsharp plugin](http://ionide.io/). And there are several vi plugins available for VS Code ([I use this one](https://marketplace.visualstudio.com/items?itemName=vscodevim.vim)).

## Interfacing to native code

Whenever a program executes on a virtual machine (VM) of some sort, like the Java Virtual Machine (JVM), the .Net Common Language Runtime (CLR) or the Python interpreter, it is isolated from the underlying operating system, which is often a good thing. For example, the Virtual Machine might be uniquely matched to the programming language or problem domain, made more secure, be operating system independent, etc. Sometimes however, there is a requirement for a VM based application to directly use a native library, for example to gain access to some OS-specific service not provided by the VM, custom hardware, etc. These native libraries are typically written in the native programming language of the underlying operating system, or the implementation language of the VM itself. Often, this is language is C.

Here are a few examples of Virtual Machines and the corresponding interop technologies for accessing native libraries:

| Virtual Machine | Interop Technology |
| ---- | ---- |
| Java Virtual Machine | Java Native Interface (JNI) |
| .Net CLR | P/Invoke |
| Android Dalvik VM | Android Native Development Kit (NDK) |
| Python Interpreter | Simple Wrapper and Interface Generator (SWIG) |

.Net's P/Invoke mechanism is a lot slicker than many of the alternatives, because it is purely declarative; you can link to arbitrary code in arbitrary shared libraries without having to write shims or native adapter code. For each entry point (function) in the shared library, all you have to do is proivde declarations that define the data types involved, how they might need to be translated, specify the alignment etc., and you are done. You do all this in your high-level language.

Contrast this with Java's JNI, where you actually need to write a glue layer in C or C++, that handles each and every method invocation. What is worst about this is that now you have another native library to deal with (the JNI glue), which makes cross-platform portability so much harder. And if you want to dispatch a callback from C back into your JVM code, [good luck](http://stackoverflow.com/questions/4313004/registering-java-function-as-a-callback-in-c-function/4330239#4330239)! In F#, you can do the equivalent in a few lines of F# code. Wrapper generators (such as the excellent [SWIG](http://swig.org/)) can make this task easier. With .Net, this is also possible (using C#), but rarely necessary.

From a high level, interoperating between a VM language (e.g. F#) and a native languages (e.g. C) requires figuring out several issues:

* What are the library naming conventions on the OS? Where are these libraries stored? How do they get found and loaded?
* What are the names of the entry points for the functions that are contained in the library?
* What are the calling conventions, i.e. who is responsible for cleaning up the stack after the invocation, the caller or the callee?
* Is the byte sex the same on both sides (little endian/big endian)
* How do C and F# function arguments correspond? If they are not the same, how are the translated ("marshalled") into one another?
* If there are structures, how are they aligned in memory? Is there padding between individual elements?
* Who owns any memory blocks that pass the boundary? Who gets to allocate or free them?
* How do you stop important data structures on the VM from getting garbage collected?
* Is it computationally expensive to cross the boundary between VM and native code?
* How do you call back from native code into VM code? Is it OK for any thread to do this, or are there special restrictions?

P/Invoke allows you to address most if not all of those questions in a purely declarative manner, i.e. without having to write any shims or other adapter libraries. Don't be intimidated by the length of this list; once the appropriate declarations are written, it is very comfortable and easy to write code that uses native libraries!

## Our target: the RTL-SDR library

For this discussion, I will use the excellent rtl-sdr library as an example. SDR stands for Software Defined Radio, where radio communications components (such as for example mixers, filters, demodulators etc.) that were traditionally implemented in hardware, get replaced by software. What remains of the hardware radio is typically a frontend, a downconverter of some sort, some filtering, a fast analog-to-digital converter to sample the incoming radio signal into (large amounts of) numbers and a high-speed digital interface to send said bits into the computer. Sounds expensive? It used to be, until a few years ago, when [some smart people figured out](http://rtlsdr.org/#history_and_discovery_of_rtlsdr) how to take a cheap (USD 20) mass-produced USB-stick that was intended for TV reception, and turn it into a general purpose SDR radio. Not exactly stellar performance, but the price is right. A library was developed that provides access to the hardware, which is called librtlsdr. Find a detailed description [here](http://sdr.osmocom.org/trac/wiki/rtl-sdr). If you want to buy one of these devices, here is [an extensive list of compatible devices](https://www.reddit.com/r/RTLSDR/wiki/compatibility).

To get the rtl-sdr library onto your system, you can either build it from source, or install it from the package manager that comes with your distro. For example, I am on Mint 17.3 (an Ubuntu derivative), so I can just use

```
sudo apt-get install librtlsdr-dev
```

This will pull in all the relevant dependencies, notably libusb.

OK, so what did we just install? This is a C library we are talking about, so we are looking for an include file (`/usr/include/rtl-sdr.h` on my system), and a library (for me it is `/usr/lib/x86_64-linux-gnu/librtlsdr.so`). Oh, and as we are talking to external hardware, the permissions need to be set right, which is often done through `/etc/udev`. In my case, the package manager took care of this. If you have permission trouble with access to the hardware, check the udev configuration.

And while this article is about Linux, librtlsdr-dev and its dependencies are also available to install via Brew on macOS, and if you do that, all the code in this post will work the same way on your Mac. I didn't try Windows (the installation of libusb and librtlsdr is a little more involved there), but it shouldn't be too hard to get it to work on Windows also.

## Talking to librtlsdr from F#

All right, we have the libary, now let's take a look on how to access it from F#. Remember what I said above about P/Invoke being purely declarative? Let's try this out! The simplest possible function that we can identify in the `/usr/include/rtl-sdr.h` include file is the `rtlsdr_get_device_count()` function:

```
RTLSDR_API uint32_t rtlsdr_get_device_count(void);
```

What is this? The `RTLSDR_API` thing expands to an import or export statement, depending whether the library itself is being compiled or a program that uses it. From our perspective, this is equivalent to an `extern` declaration. `uint32_t` is a pretty obvious typedef. So armed with this knowledge, we can write a declaration to make this function callable from F#:

```
[<DllImport("librtlsdr", CallingConvention=CallingConvention.Cdecl)>]
extern uint32 rtlsdr_get_device_count()
```

The `[<Stuff>]` is called an Attribute. This specific attribute is called DllImportAttribute. DLL is Windows-speak for a shared library, otherwise known as a .so (or .dylib if you are on macOS), "librtlsdr" is obviously the name of the library, and the calling convention specifies who is responsible for cleaning up the stack after the call.

Let's try and write a tiny little F# program that loads the C library, and sees whether we have any RTL-SDR devices attached. I am assuming you have .Net Core installed (the RTL version, right?), as well as the rtlsdr library. As long you have those two prerequisites, it shouldn't matter whether you are on Linux, macOS or Windows, we all value diversity! Now create an empty directory, cd into it, and do the following:

```
dotnet new --lang fsharp
dotnet restore
dotnet build
dotnet run
```

This will do a few things:

* create two new files: `Program.fs` and `project.json`
* fetch any dependencies
* build an executable
* run it.

If this says "Hello World!", you are in business (and yes, they forgot the comma between "Hello" and "World". Get a life!). Now open `Program.fs` in an editor and replace the code in that file with the following:

```fsharp
open System.Runtime.InteropServices

[<DllImport("librtlsdr", CallingConvention=CallingConvention.Cdecl)>]
extern uint32 rtlsdr_get_device_count()

[<EntryPoint>]
let main argv = 
    rtlsdr_get_device_count() |> printfn "Found %A device(s)"
    0
```
	
Hang on - what's with that syntax? OK, line 1 is some boring import/use/open/whatever. We talked about lines 3-4 before, this is just the declaration that tells the system how to call the function `rtlsdr_get_device_count()` in the library "librtlsdr". The EntryPoint is pretty obviously the main function, starting in line 7.

You already know what the funny `|>` in line 8 is. No, believe me, you do. It is called a pipe. As in `ls | grep foo`. In this case, we send the output of the `rtlsdr_get_device_count()` into the function `printfn` (as its last argument, to be precise). This is really easy to get used to!

Now save `Program.fs`, and issue another

```
dotnet run
```

In response, it will hopefully say something like:

```
Project tmp (.NETCoreApp,Version=v1.0) will be compiled because inputs were modified
Compiling tmp for .NETCoreApp,Version=v1.0

Compilation succeeded.
    0 Warning(s)
    0 Error(s)

Time elapsed 00:00:07.3866507

Found 1u devices
```

Well, that was easy! The machinery was able to figure out that you need to build things before running them, and depending on whether or not you have one of those nifty USB devices in your computer, it will report 0u or 1u devices. But what is a `1u`? This would be a good time to talk about types.

You wouldn't know it from looking at the source code, but F# is a strongly typed language, and I mean *strongly*. It will not take an unsigned integer in place of a signed one, or make an int into a float until you explicitly instruct it to do so.

But if this a typed language, where are the wordy annoying type declarations in the code above? Does `main` have a type, and if so, what is it? Well, `main` is of the type `string[] -> int`, which means that it is a function that takes an array of strings and returns a signed 32-bit integer. And in fact, the seemingly unmotivated `0` in line 9 is the return value of `main`. You can verify this by replacing the `0` with a string `"foo"`, and see what the compiler says: It will yell at you for trying to return a string, where an integer was required.

One of the nice things about F# is that you don't have to constantly remind it of the types of everything - types are inferred (I call it "types without the typing"). What this means is that the compiler can usually figure out the types by itself, and check for consistency. If it can't, it will tell you with an error message, and usually a simple type annotation or two is all the guidance it needs. You get all the benefits of strong typing (compile-time or even edit-time error checking) without the cost (having to type lots of redundant stuff).

To put it slightly differently: You don't need to say for the 157th time

```
public static void Main(string[] args)
{}
```

because

```
public static void Main(string[] args)
{}
```

is always

```
public static void Main(string[] args)
{}
```

wie es singt und lacht. Let the damn computer figure it out.

In F# a lot of the defaults are the "other way round" from what you may be used to. For example, variables aren't. Once you assign a name to a value by saying `let a = 3`, `a` will have the value of 3 until the end of time. And no, that is not a problem, it just takes some getting used to. If you still want `a` to be a "variable", you need to explicitly say so. If you tend to "forget" to look at the return value of function calls, the compiler will politely remind you to pipe the return value to the `ignore` function. You can still keep all your bad habits, you just need to be explicit about them. After a while you might realize that is actually much less work to do the Right Thing, and presto - you have discovered "long-term laziness": Doing what is the least amount of work in the long run.

## Coming attractions

If you are still with me, you may ask yourself, "why bother with all this, and not write this code directly in C?" Excellent question, simpler is almost always better. What F# gives you is the ability to work on higher levels of abstraction. Often what you end up with is code that is substantially easier to read and understand than C (and also to write, with a little practice).

Here's another lame analogy for you: Imagine cooking a meal in "cooking show style"; rather than having some semi-hostile looking vegetables to deal with, every ingredient is nicely peeled, chopped, and available for use in its own little glass bowl. In my mind, using native libraries from F# is like that: All the ingredients are conveniently arranged for you to be creative with. True, you have to deal with all the dishes (writing P/Invoke declarations), but once you have done that, the process is so much more enjoyable, which often leads to better outcomes.

So in the future, please come back to read about the following attractions that I didn't have time to cover today:

* Units! dB, even!!
* The Hollywood Principle, written in two lines of code!
* Agents! (non-Hollywood)
* Asynchronous stuff!
* Invisible error handling!
* Listen to your favorite FM station using artisan F# code

For comments, feel free to [hit me up on Twitter!](https://twitter.com/janschiefer) And in the meantime, you can order yourself one of those great cheapo USB radios!

## Where to go for more information

The intersection of F#, P/Invoke and .Net Core is currently a little under-documented. Here are some references I find useful:

* [Microsoft on native interop in dotnet core](https://docs.microsoft.com/en-us/dotnet/articles/standard/native-interop)
* [Lots of detail, focused on Mono, but all the mechanisms are the same](http://www.mono-project.com/docs/advanced/pinvoke/)
* [More advanced: The obscure world of references in F#](http://stackoverflow.com/questions/5028377/understanding-byref-ref-and?rq=1)
* The book [Expert F# Programming](http://www.apress.com/us/book/9781484207413) has a great chapter on F# and P/Invoke
* [MSDN article on F# and P/Invoke](https://msdn.microsoft.com/en-us/library/hh304361(v=vs.100).aspx)

Other projects that have more of a focus on C# and/or Windows:

* The [pinvoke project](https://github.com/AArnott/pinvoke), which collects P/Invoke definitions from various libraries
* what seems to be an "official" [P/Invoke library for RTL-SDR](https://github.com/librtlsdr/SharpRTL) in C#.

No Monads were harmed in the creation of this blog post.



# Using OpenCL on F# via FSCL #

*All text and code copyright (c) 2016 by Jose Mª González. Used with permission.*

*Original post dated 2016-11-29 available at https://josesoyo.wordpress.com/2016/11/29/using-opencl-on-f-via-fscl/*

**By Jose Mª González**


As user of F# I love the language but today I'd like to talk about the point that I've been struggling more, how to run F# code on GPU.
On [Fsharp.org](http://fsharp.org/use/gpu/) there's a nice list of libraries that can run F# on GPU, but some of them are very old
and specially they are not updated since long time ago.  Another problem that you can find is that there's few documentation.
For this reason I decided to write a simple example on how to use FSCL (Fsharp to OpenCL), one of the methods to run F# code on GPU.

FSCL can be installed from NuGet and since I prefer to work with scripts, the first thing that you must do it's to reference all the library files and open FSCL

```fsharp
// How to reference FSCL - It's very important the order
#r @"../packages/FSCL.Compiler.2.0.1/lib/net45/FSCL.Compiler.Core.dll"
#r @"../packages/FSCL.Compiler.2.0.1/lib/net45/FSCL.Compiler.dll"
#r @"../packages/FSCL.Compiler.2.0.1/lib/net45/FSCL.Compiler.Language.dll"
#r @"../packages/FSCL.Compiler.2.0.1/lib/net45/FSCL.Compiler.NativeComponents.dll"
#r @"../packages/FSCL.Compiler.2.0.1/lib/net45/FSCL.Compiler.Util.dll"
#r @"../packages/FSCL.Runtime.2.0.1/lib/net451/FSCL.Runtime.CompilerSteps.dll"
#r @"../packages/FSCL.Runtime.2.0.1/lib/net451/FSCL.Runtime.Core.dll"
#r @"../packages/FSCL.Runtime.2.0.1/lib/net451/FSCL.Runtime.Execution.dll"
#r @"../packages/FSCL.Runtime.2.0.1/lib/net451/FSCL.Runtime.Language.dll"
#r @"../packages/FSCL.Runtime.2.0.1/lib/net451/FSCL.Runtime.dll"
#r @"../packages/FSCL.Runtime.2.0.1/lib/net451/FSCL.Runtime.Scheduling.dll"
#r @"../packages/FSCL.Runtime.2.0.1/lib/net451/OpenCLManagedWrapper.dll"

open FSCL.Compiler
open FSCL.Language
open FSCL.Runtime
```

The library contains a number of functions to see which ones are your OpenCL devices and it's useful to check before you execute
any code if the OpenCL drivers are already installed and OpenCL is ready to run, so before starting we ask to the system how many OpenCL devices we have

```fsharp
open OpenCL 

// return the number of OpenCL devices present in your system
OpenCLPlatform.Platforms.Count 
```

Thus, after we have called FSCL and we know that our device is compatible with OpenCL, we can start defining constants and functions which are very similar to the standard ones

```fsharp
[<ConstantDefine>] 
let pi = 
    3.1415926f

// define functions:

// FSCL function
[<ReflectedDefinition;Kernel>]
let DoStuffOpenCL (a1:float32[]) (a2:float32[]) (b:float32[]) (c:float32[]) (iters:float32[]) (wi:WorkItemInfo) =
    // a1 and a2 are the same length
    // b, c and  iters are the same length and always lower than the length of a1&a2
    let gid = wi.GlobalID(0)
    let outArr = Array.zeroCreate<float32> a1.Length
    let mutable acc = a1.[gid]

    for i in 0..iters.Length-1 do
        acc <- b.[i]*cos(pi*iters.[i]*a2.[gid]+c.[i])+acc
    outArr.[gid] <- acc
    outArr

// Fsharp function
let DoStuff (a:float32, b:float32[], c:float32[], iters:float32[])  =
    // function to compute the globlal modulation (sum of all)       
    Array.fold(fun acc x ->  acc+b.[x]*cos(pi*iters.[x]*a+c.[x]) ) 0.f [|0..iters.Length-1|]  // value*b.[i]*cos(pi*iters.[i]*a2.[gid]+c.[i])+acc
```

As you can see, a function ready to run in OpenCL with FSCL is very similar to one that you'll write normally with:

* Attributes: ReflectedDefinition and  Kernel. ReflectedDefinition is necessary to specify it on all the FSCL functions and Kernel if it's a Kernel.
  it's not always necessary to use the attribute kernel since nested functions are working.

* the parameter called WorkItemInfo that we need to pass to all our FSCL Kernel indicating the global and local size of our kernel.

In this case, both functions used properly will produce the same output, so now we need to define some arrays to operate with them and which one is the size of our kernels:

```fsharp
let ws = WorkSize(262144L)       // 2^18 

// Define the long Arrays
let A1 = Array.zeroCreate<float32> 262144
let A2 =  Array.zeroCreate<float32> 262144 |> Array.mapi(fun i _ -> float32(i))

// Define the short arrays
let amplitudes  = [|0.2f;0.5f;0.35f|] 
let frequencies = [|1.2f;10.f;52.5f|]   // this will be the iters
let phases      = [|pi;pi/4.f;1.5f*pi|] 
```

The last step is to run the kernel and compare how fast is compared with the Fsharp equivalent function and we iterate it many times to see the performance:

```fsharp
#time    

// CPU
[|0..1000|] |>
Array.iter(fun _ -> 
(A1 ,A2) ||> Array.map2(fun old t -> old+(DoStuff( t, amplitudes, phases, frequencies))
                           ) |> ignore
         )


// OpenCL
[|0..1000|] |>
Array.iter(fun _ -> <@ (DoStuffOpenCL A1 A2  amplitudes phases frequencies ws) @>.Run() |> ignore )
```

Here I show the CPU serial code, but it can be run in parallel with the use of `Array.Parallel.iter()`. The comparatives for my laptop (i7+Gforce GTX 960M) are:

```
FSCL:                | Real: 00:00:11.043, CPU: 00:00:08.406 |
F#:                  | Real: 00:03:21.406, CPU: 00:03:21.812 | 
Array.Parallel.iter: | Real: 00:00:55.536, CPU: 00:04:54.671 |
```

So using the GPU with FSCL it's been possible to accelerate the code x5 with respect to the parallel way on CPU,
but it's important to remember that since the code is converted to OpenCL, the use is not restricted to GPU and can be run in many devices,
as an example, it also runs well on [Intel's Xeon-Phi](http://www.intel.com/xeonphi).
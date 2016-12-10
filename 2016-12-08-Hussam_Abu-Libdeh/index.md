
# How F# delighted this newbie while experimenting with distributed systems #

*All text and code copyright (c) 2016 by Hussam Abu-Libdeh. Used with permission.*

*Original post dated 2016-12-08 available at https://hussam.github.io/fsadvent16/*

**By Hussam Abu-Libdeh**

A few months ago, I was researching routing and load balancing strategies in distributed systems.
Given a set of servers and a stream of incoming client requests, how should the requests be spread across the different servers?
(assuming that any server can handle any request). Intuitively, the answer depends on a multitude of factors such as whether requests
are CPU or IO bound, the existing load at each of the servers, the processing power of the different servers, whether we are optimizing
for response latency or throughput, whether we can cache results of previous requests, and many more complications that make this deceptively
easy question hard to answer in practice.

In situations like this, it is often helpful to reduce the problem to a simpler version that allows us to poke and play with it
until we're comfortable adding more complications back into the mix. So, here's a very simple version of this problem:
assume that all the servers are identical, and a client's request simply makes the server thread sleep for a few milliseconds
depending on a "job size" defined in the request payload. Each server can only "process" one request at a time, and while a
server is processing a request job, it buffers incoming requests in a work queue for future processing. Client requests are
funneled through a load balancer that periodically probes the length of the work queue at each of the servers. Client requests
have different job sizes, but the load balancer isn't aware of them. The load balancer routes requests to servers in the order
it receives them, and its goal is to minimize the aggregate queueing delay that's experienced by client requests.

## F# Experiment Harness

For F# 2016 Advent Calendar, I wanted to share my prototype code that I used to play with this problem and highlight a couple of things
that I thought were pretty cool. The code is available [here](https://github.com/hussam/fsadvent16). The program instantiates a set of servers and clients that communicate using UDP,
servers process requests using a `MailboxProcessor`, and the experiment harness runs the clients using different configurations and plots
the results using [F# Charting](https://fslab.org/FSharp.Charting/fsharpcharting.html). The things I want to highlight are not necessarily sophisticated, but as an F# newbie, I thought they were pretty damn cool!

### Cool thing #1: Async Workflows

Expressing the UDP sending and receiving loops of clients and servers was incredibly easy using asynchronous workflows.
I could run these workflows in parallel, and even have the receive loop collect results and return them. Wooot! For example, here's the code for my client:

```fsharp
let sender = async {
   use socket = new UdpClient()
   for i in 1..msgsToSend do
      // Pick the server to which the request will be forwarded
      let targetIndex, probabilityOfSelection = routingFunc(queueLengths)
      let serverHostname, serverPort, _ = queueLengths.[targetIndex]

      // Send the message to the server and measure the extra delay
      let jobSize = rand.Next(minJobSize, maxJobSize)
      let sendTime = timer.ElapsedMilliseconds
      let msg = Array.concat [| BitConverter.GetBytes(jobSize); BitConverter.GetBytes(i); BitConverter.GetBytes(port) |]
      socket.Send(msg, msg.Length, serverHostname, serverPort) |> ignore

      jobsInFlight.[i] <- (featurize(queueLengths), jobSize, targetIndex, probabilityOfSelection, sendTime)
   return None
}

let receiver = async {
   use socket = new UdpClient(port)
   let results = new List<_>()
   let anySender = new IPEndPoint(IPAddress.Any, 0)
   while results.Count < msgsToSend do
      let bytes = socket.Receive(ref anySender)
      let endTime = timer.ElapsedMilliseconds
      let jobId = BitConverter.ToInt32(bytes, 4)

      let (qlens, jobSize, selectedServerIdx, p, sendTime) = jobsInFlight.[jobId]
      let delay = int(endTime - sendTime - int64(jobSize))
      results.Add((qlens, jobSize, selectedServerIdx, p, delay))
   return (Some results)      // return the delays experienced
}

let results =
   [receiver; sender]
   |> Async.Parallel
   |> Async.RunSynchronously
   |> Array.head
```

### Cool thing #2: Pipe Magic

The forward pipe operator `|>` really helps make the code concise and highly readable. Here is the code that runs a number of clients in parallel,
waits for them to finish, collects their results, sifts through them, builds a CDF of message delays, and graphs it.

```fsharp
[| 1..config.numClients |]
|> Array.map(fun i -> async { return Client.Run( (* client params *) ) } )
|> Async.Parallel
|> Async.RunSynchronously
|> Array.fold (fun accIn clientResults -> Array.append accIn (clientResults.ToArray())) [||]
|> Array.map (fun ((_, _, _, _, experiencedDelay)) -> experiencedDelay)
|> Array.sort
|> Array.mapi (fun i latency -> (100.0 * float(i+1) / float(config.msgsToSend), latency))
|> Chart.Line
```

The full test harness was not much bigger than this code above; there were a few lines of code to instantiate server processes
and repeat the experiment a few times under different configuration parameters, but that was it! Running experiments in a few lines of code is pretty damn awesome!

### Cool thing #3: Start Running Quicker and with Fewer Errors

F# is a strongly typed language with static type inference. This really helps me as a programmer not mess up -- and I need that because writing
and debugging distributed systems is hard enough as it is. Processing a list or an array? The type system will make sure you think about what happens if it is empty.
Processing something that may or may not contain a value? Option types force you to think about both cases.
Calling a function that returns a non-unit value? You have to explicitly state when you're ignoring the returned value.
This resulted in fewer errors and a quicker time to play and experiment with my system.

All in all, yay F#! 


Source code for this article: [Tar.gz](https://github.com/hussam/fsadvent16/tarball/master), [Zip](https://github.com/hussam/fsadvent16/zipball/master)
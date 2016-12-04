
# NeuralFish - Evolving Neural Topologies in F# #

*All text and code copyright (c) 2016 by Jeremy Bellows. Used with permission.*

*Original post dated 2016-11-30 available at http://jeremybellows.com/blog/NeuralFish-Evolving-Neural-Topologies-In-FSharp*

**By Jeremy Bellows**

## Preface

A few months ago, I had the pleasure of reading the ["Handbook of Neuroevolution Through Erlang" by Gene Sher](http://amzn.to/2fYJDR9).
To sum up my experience, the book changed the way I perceive problems. The knowledge I gained from the book allowed me to explore valuable perspectives during introspection, leading to ideas and solution's previously inaccessible.

Excited about the ideas presented in the book, I built [NeuralFish](https://github.com/JeremyBellows/NeuralFish).
NeuralFish is based on the [DXNN system](https://arxiv.org/abs/1008.2412). It should be considered a prototype at this time (11/2016).
In this blog post, we'll explore some of the concepts NeuralFish is built off of and create a script that uses NeuralFish to evolve neural topologies that are able to identify SMS spam.

I plan on exploring ideas and concepts that NeuralFish operates on in future blog posts.
Follow me on Twitter [@JeremyBellows](https://twitter.com/JeremyBellows) or [connect with me on LinkedIn](https://www.linkedin.com/in/jeremybellows) to keep updated.

## Introduction

Reality operates on a set of observed and unobserved rules. Natural systems have developed to exist within the entropy of reality,
obeying these rules involuntarily. These natural systems build off of each other, often creating complexity that is soon culled by the unobserved rules.

The earth is covered with life that is capable of existing in the systems that reality dictates. Life that is able to consistently solve encountered problems. The fundamental function that allows life to be successful is replication. Through replication life is able to mutate. Mutated life is composed differently from previous generations giving it either a disadvantage or advantage in its ability to pass on traits. This concept defines an evolving system. The goal of NeuralFish is to use these concepts to evolve artificial intelligences that have an advantage in the configured programmed reality.

## NeuralFish

Using the fundamental ideas of evolution, it is possible to create artificial intelligences capable of solving encountered problems.
NeuralFish defines neural topologies as the type `NodeRecords`, which is a map of `Sensors`, `Actuators`, and `Neurons`.
During the evolution process, `NodeRecords` are mutated from a set of mutation operators, reconstructed into an active set of neural processes,
fed sensory data which is processed by the NeuralNet activating respective actuators, and then scored.
Scored records are accumulated and then culled to only allow the 'fittest' solutions to mutate.
This process is repeated for a configurable number of generations.

Mutating Neural Networks allows for sensors or actuators to be added and learned, neuron connections to be formed, and more.
This mechanism allows for discovery and learning of potential solutions.

The prototype is designed to solve single-scope problems, i.e. the constructed problem scape consists of a single problem.
Problem scapes define the rules of the reality that dictates the evolving solutions existence. Problem scapes are defined in NeuralFish by configurable variables (and functions).

The goal of the prototypes is to demonstrate setting up problem scapes in fsharp, thinking through how to 'score' the fittest solution,
and produce neural topologies evolved under configured parameters.

## SMS Spam Detection


### Objective

Create an artificial intelligence that is capable of reading text messages and determining if the message is spam or legit (henceforth referred to as Ham).

### Getting Started

In the spirit of simplicity, the prototypes are built as scripts and placed in a folder called `temp`. This allows loading of the code in an interactive environment.

Clone the NeuralFish repository by executing this command in git bash.

```
SSH
git clone git@github.com:JeremyBellows/NeuralFish.git
```

```
HTTPS
git clone https://github.com/JeremyBellows/NeuralFish.git
```

### The temp folder

The `temp` folder has been added to the `.gitignore` file. Any files added in that folder will be ignored by git.

Create a folder called `temp` in the root of the repository.

Create the file `SMSSpamDetector.fsx`

Download the [SMS spam collection data set zip](http://mlr.cs.umass.edu/ml/machine-learning-databases/00228/) and unzip the contents into the `temp` folder.
This data is available via the [UMass Machine Learning library](http://mlr.cs.umass.edu/ml/datasets/SMS+Spam+Collection).

### Loading Dependencies

NeuralFish has been packaged in an interactive script for simple dependency loading. This file can be found at [NeuralFish/NeuralFish_dev.fsx](https://github.com/JeremyBellows/NeuralFish/blob/d5f3af4114b0e1eb249d9f275f5eede35349d9cd/NeuralFish/NeuralFish_dev.fsx)

In `temp/SMSSpamDetector.fsx`, load the NeuralFish interactive script into memory.

```fsharp
#I "../NeuralFish"
#load "NeuralFish_dev.fsx"
```

The following dependencies will also need to be opened for access to types and functions

```fsharp
open NeuralFish.Types
open NeuralFish.Core
open NeuralFish.EvolutionChamber
```


### Loading SMS Data

The SMS Data contained in `SMSSpamCollection` is composed of the correct answer (Ham|Spam) followed by 4 whitespaces and the SMS contents.

The first part of the SMS collection is composed of 2 possible options, Ham or Spam. These will need to be pattern matched often so it's best to express this using the type system.

```fsharp
type MsgType =
  | Ham
  | Spam
```
  
NeuralFish contains types and functions for training single-scope problems such as this one.
[`TrainingAnswerAndDataSet<'T>`](https://github.com/JeremyBellows/NeuralFish/blob/d5f3af4114b0e1eb249d9f275f5eede35349d9cd/NeuralFish/Types.fs#L216)
is used to model the data for consumption.
The raw type is `('T*(float seq)) array` where `'T` is the correct answer for the data in the tuple.

The next block of code utilizes pattern matching and type safety to load the data into a usable format.

```fsharp
let data : TrainingAnswerAndDataSet<MsgType> =
  let convertDataIntoVector (hamOrSpam, data) =
    let convertDataIntoVector (data : string) =
      data
      |> Seq.map (fun x -> float x)
    (hamOrSpam, data |> convertDataIntoVector)
  let splitDataIntoTuple (data : string) =
    let matchHamOrSpam (hamOrSpam : char array) =
      match hamOrSpam with
        | [|'h';'a';'m';_|] | [|'h';'a';'m'|] ->
          Ham
        | _ ->
          Spam
    let msgType =
      data
      |> Seq.take 4
      |> Seq.toArray
      |> matchHamOrSpam
    let StringRemoveFirstFour (sequence : string) =
      sequence
      |> Seq.rev
      |> Seq.take ((sequence |> Seq.length) - 4)
      |> Seq.rev
    (msgType, data |> StringRemoveFirstFour |> System.String.Concat )

  sprintf "%s/SMSSpamCollection" __SOURCE_DIRECTORY__
  |> System.IO.File.ReadAllLines
  |> Array.Parallel.map splitDataIntoTuple
  |> Array.Parallel.map convertDataIntoVector
```
  
### Interpreting NeuralNet Output

A NeuralNet is a map consisting of `Sensors`, `Neurons`, and `Actuators`. During a think cycle,
the `Cortex` synchronizes all sensors in the NeuralNet, waits for NeuralNet activity to stop, then activates the actuators.

The single-scope problem of SMS Spam detection only needs one actuator. This actuator will guess whether the SMS contents are Spam or Ham.

Define the Id of the Spam Detect actuator for NeuralNet output lookup.

```fsharp
let spamOrHamOutputHookId = 0
```

Using a cortex to synchronize input and output means that the result of a think cycle will always be a complete map of actuator output.
The `InterpretActuatorOutputFunction<'T>` type is used to interpret NeuralNet actuator output during single-scope training.

The initial way I solved this problem was by providing a binary method of interpretation.

```fsharp
let (convertNetworkOutputToHamOrSpam : InterpretActuatorOutputFunction<MsgType>) =
  (fun networkOutputMap ->
    match networkOutputMap |> Map.tryFind spamOrHamOutputHookId with
    | Some networkOutput ->
      if (networkOutput > 0.99) then
        Ham
      else
        Spam
    | None ->
      NotSure
  )
```
  
Since the intelligence is evolved, then there is no simple way of determining how it is able to perform its task. Therefore, it is necessary to give the intelligence a reality where it is forced to be precise.

Under this reality, the possible answers the artificial intelligence can give should include uncertainty. The desired evolved topology should not answer with uncertainty often, so it should be disfavored during NeuralNet scoring.

```fsharp
type MsgType =
  | Ham
  | Spam
  | NotSure
```

```fsharp  
let (convertNetworkOutputToHamOrSpam : InterpretActuatorOutputFunction<MsgType>) =
  (fun networkOutputMap ->
    match networkOutputMap |> Map.tryFind spamOrHamOutputHookId with
    | Some networkOutput ->
      if (networkOutput > 0.99) then
        NotSure
      else if (networkOutput < 0.01) then
        Ham
      else
        Spam
    | None ->    
      NotSure
  )
```
  
### Scoring the Interpreted Neural Output

The training function for single-scope problems that NeuralFish exposes is capable of keeping track of the correct answers that correspond to sensory data.
It exposes this capability through the type [ScoreNeuralNetworkAnswerFunction<'T>](https://github.com/JeremyBellows/NeuralFish/blob/d5f3af4114b0e1eb249d9f275f5eede35349d9cd/NeuralFish/Types.fs#L222).

After NeuralNet think cycles are finished, scores are accumulated and summed to represent the total score of that topology.
The `ScoreNeuralNetworkAnswerFunction<'T>` is arguably the most important configuration of this prototype as it will determine which toplogies are mutated.

Since the desired artificial intelligence should value precision, then the scoring should implement positive and negative reinforcement to guide the evolution process.

The desired artificial intelligence for this job should never identify actual messages as spam. If this program were to be implemented into a live production environment, then the user experience should not be impacted by this implementation. Therefore it is important to give positive reinforcement on actions that provides a better user experience and negative reinforcement on actions that diminishes the user experience.

```fsharp  
let (scoreAnswer : ScoreNeuralNetworkAnswerFunction<MsgType>) =
  (fun correctAnswer guessedAnswer ->
    if (guessedAnswer = correctAnswer) then
      match correctAnswer with
      | Spam ->
        1.0
      | Ham ->
        2.0
      | NotSure ->
        0.0
    else
      match correctAnswer with
      | Spam ->
        -1.0
      | Ham ->
        -2.0 
      | NotSure ->
        0.0
  )
```
  
### Evolving the NeuralNet

Evolution and training properties are defined as typed records. These records are passed to exposed `NeuralFish.EvolutionChamber` functions that process evolution. Default properties are available for experimentation.

* [Training Properties](https://github.com/JeremyBellows/NeuralFish/blob/d5f3af4114b0e1eb249d9f275f5eede35349d9cd/NeuralFish/Types.fs#L224-L240)
* [Evolution Properties](https://github.com/JeremyBellows/NeuralFish/blob/d5f3af4114b0e1eb249d9f275f5eede35349d9cd/NeuralFish/Types.fs#L193-L207)

The prototype utilizes the single-scope training function to perform evolution. The following code creates a record with the necessary configurations and then processes the single-scope training.

```fsharp  
let scoredRecords =
  let trainingProperties =
    let outputHookIds = [spamOrHamOutputHookId]
    let activationFunctions =
      Map.empty
      |> Map.add 0 sigmoid
    let learningAlgorithm = NoLearning
    let defaultTrainingProperties = getDefaultTrainingProperties data convertNetworkOutputToHamOrSpam scoreAnswer activationFunctions outputHookIds learningAlgorithm defaultInfoLog
    { defaultTrainingProperties with
          ShuffleDataSet = false
          MaximumMinds = 25
          MaximumThinkCycles = 25
          AmountOfGenerations = 50
          DividePopulationBy = 5
    }
  trainSingleScopeProblem trainingProperties
```
  
`activationFunctions`, `outputHookIds`, and `learningAlgorithm` defines how NeuralNets evolve and operate.
Activation Functions are all the possible ways neurons can activate. Output Hooks are used for actuators.
The learning algorithm is a configurable option that allows for neural plasticity.

`MaximumMinds`, `MaximumThinkCycles`, `DividePopulationBy`, and `AmountOfGenerations` configures the evolution of NeuralNets.
`MaximumMinds` sets how many NeuralNets should be created in a generation.
`MaximumThinkCycles` dictates the lifetime of the NeuralNet and how many think cycles it goes through.
`DividePopulationBy` will sort the scored NeuralNets then divide the array by the supplied number. ([There is an issue open to change this](https://github.com/JeremyBellows/NeuralFish/issues/47)). 
`AmountOfGenerations` limits the amount of generations that should be iterated through before returning the result.

The `evolveFromTrainingSet` function uses provided training properties to evolve NeuralNets starting with the configuration `StartingRecords`.
Specifically the function is a wrapper on top of [evolveForXGenerations](https://github.com/JeremyBellows/NeuralFish/blob/d5f3af4114b0e1eb249d9f275f5eede35349d9cd/NeuralFish/EvolutionChamber.fs#L458) which exposes further configuration for advanced evolution.

The training occurs with NeuralNet activity logged via the `InfoLog` function. The `defaultInfoLog` function is set to print to the console.

Open a command prompt, navigate to the `temp` folder, and execute

```
fsi SMSSpamDetector.fsx
```

*With the set configuration above it may take a awhile for training to finish. Shorten the think cycles, number of generations, or maximum minds to reduce run time*

The final generation should have their scores printed to the console. This demonstrates the evolution process occurring but does not demonstrate the solutions ability to solve problems in our reality. To do so would require exposing the NeuralNets to a variety of problems over a large number of generations. It's also worthwhile to experiment with activation functions and learning algorithms.

[Watch a video of recorded evolution!](https://www.youtube.com/embed/2ttXxpbn52Y)


## What's Next?

NeuralFish will be capable of much more than single-scope problems. As the [DXNN system](https://arxiv.org/abs/1008.2412) has demonstrated, there are many uses for this technology.

We live in a world that is constantly changing. Every step towards the Automation Era brings fantastical ideas to life like:

* self-driving cars
* gadgets capable of discovering power sources
* programs that learn how to efficiently interact with users

What comes next is left to human imagination!
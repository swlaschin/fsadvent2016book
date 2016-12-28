



# Advent with a Star #

*All text and code copyright (c) 2016 by Carsten König. Used with permission.*

*Original post dated 2016-12-23 available at http://gettingsharper.de/2016/12/23/advent-with-a-star/*

**By Carsten König**

![](graphics-christmas-reindeer-808473.jpg)

It’s that time of the year again where F#ers come together and write a bit about
what they enjoy.

I really enjoyed this years [Advent of code](http://adventofcode.com/2016) and
while I only did a few of the exercises in F# I want to write about one
algorithm that helped out in quite a few of the exercises.

The Algorithm I’m talking about is the
[A* algorithm](https://en.wikipedia.org/wiki/A*_search_algorithm).

## Ok what is this

It’s a very general algorithm that helps you find an *optimal* path
in some kind of graph and so it’s often corelated with path-finding
algorithms.

And indeed one of the *AoC* days ([day 13](http://adventofcode.com/2016/day/13))
was all about finding the shortest Path in a maze.

Here is what the program I’m gonna show you here found for my input to this day:

```
#..###.#...#..#....##.#...#....#....#..#.#.##...##
#O#..#.##.#######.#.#.####.##.###...##.#..#.###...
#OO#.#.##.#....##...#..#.####.####...#..#..#..#.#.
##OO##.##.#..#...###.#.....#.....##.######....#..#
..#O##..########.#.#####...#..##.##.#.....##.###.#
#..OOOOOOOOO.###.....#####.#####....#..##..#.###..
###.##.####O..#.##....#..#.#..#.##.####.##....####
#.#..#.##.#O#.######..#.##..#.####..#.#######..###
.###......#OO#..#..####.###..#..#.#.##..#.......##
.####..###.#O##.#.#...#..#.#.##.##......#..####..#
..######.###O.###..##.#..###..##.#..##.####...#..#
#..##..#.OOOO..###.#..###......#.#####..#.#...#...
.......##O##.#.....#....#.##.#.###..#.#.####..##.#
#####.#.#O##...###.##...##.#..#..##.##...#######..
...##..##O#.###..#.###.#.##.#..#..##.#.....#..####
.#.###.#OO#...#.##...#..#.#..#.......##..#.##..##.
##..#..#O######.#####.#..###..##.###.####...#.....
....##.#O##..#...##.#.##.####..#.###....#...#..###
##...#..OOOO.#......#.....#.###...########.#####.#
###.#######O########.##.#.###.##...##..#.#.##..#.#
.##.#...###O##OOO#.##.#..#.....##....#.##......#.#
.##.#.....#OOOO#OO#.##.#.####.#.#.##..#.#####.##.#
...###.##.#.###.#O##.###...##..##.#.#......##.#..#
.#..##.#..##..##.O.#.....#..##.#..#..##..#.##.#.##
###....#...###.#.O.##.#####.#..#..##.#####..###.#.
.#.##.###.#..#.##O#.###..#..##.#####..........#.##
.####..##..#.#.##OO#OOOOO#...#.#..#####.##.##.##.#
...#.#...#..##..##OOO###O##.##..#..##.#.##.#...#.#
#..##.##..#.###.#.####.#O##.###.#.....#....#.#.###
##..#######..#..#.....##OOOO.#..######.##..#..#...
.###..#......#..###.#.#.###OO#.....#.##.###.#..###
#..##.#..########.###.###.##O##..#.#..###.#..#.##.
.#...####....#.......#...#.#O#####.#.#...###.....#
...#....#..#.#.##..#.###..##O##...##.###.####.##..
############.#.#####....#.#.OOOO#.##......#.#.#.##
#..##..#....##..#....##.#.#..##O#....##.#.##..#.##
#....#.##.#.#.#.#.###.#..######O#.###.#.##.#..#...
##.#..#.#.#.##..##..##.#.....#OO###.##...#.###.##.
.##.#...#.##.#...#.#.######..#O#..##.#.#.#.#.#####
#.#..###...#.##.##.##..##.#..#OO...#.#..##.....###
.###.#.#.#.####.##..#.....##..##...#..#.####....##
.###..##..#...#.....#..######..##.####...######..#
..#.#.#.#..##.#.##.#####..#.##.##.##.#....#...#...
```

The path it’ll find are the O‘s – if you run code
([available here](https://github.com/CarstenKoenig/AstarDemo))
you will get it with some colors too.

By the way the solution runs in about 200ms on my system which without even
*memoizing* the wall-positions.

Day 13 was not the only day where you could use this algorithm – there where
many more (the famous Day11 comes to mind).

## How does the algorithm work

It’s a *basic breadth-first search algorithm* which means that it will put nodes
it should look into in the future in a queue in such a way that it will first
search in notes on the same level, which has the big advantage that these
way to search will find always find a shortest path first.
The problem is that node-count you have to look at a level can become
really **huge** fast.

This is where `A*` shines: it uses a **heuristic** (just a functions you have to
provide that will *guess* the distance/cost to the goal and then will first
look at the nodes where this is minimal.

As most of these algorithms it uses a `visited` set as well to mark nodes
it already looked at (so you don’t revisit nodes if there are loops in the
graph).

## some details

The algorithm as I choose to implement it uses this structure:

```fsharp
type private Runtime<'node when 'node : comparison> =
    {
        heuristic : 'node -> Score
        neighbours : 'node -> 'node seq
        isGoal : 'node -> bool
        distance : 'node -> 'node -> Score
        visitedNodes : Set<'node>
        openNodes : Priority<'node>
        gScores : Map<'node,Score>
        fScores : Map<'node,Score>
        cameFrom : Map<'node,'node>
    }
```
    
to keep track of the search.

* `heuristic` is the mentioned function provided by the user (based on the
  problem) that should guess the distance/cost to the goal-node. It’s important
  that you don’t **overestimate** the cost (better keep really conservative).
  Because if you overestimate the algorithm will degenerate into a depth-first-
  search and it’s likely that you don’t find a optimal solution (as the algorithm
  stops at the first found node that is recognized as a goal).
* `neighbours` create the neighbours (children) of a node and is user-provided
  too – the algorithm uses this function to generate the next nodes.
* `isGoal` is user-provided and is used to decide if a node is a goal (the
  algorithm will stop at the first such node it finds)
* `distance` is the final user-provided function – it calculates the distance
  or cost between two nodes. The algorithm uses this only for a child and its
  parent and in the example bellow it’s just constant 1.
* `visitedNodes` is the set of nodes the algorithm already visited – no node in
  here will be revisited.
* `openNodes` is the set of nodes the algorithm can look at next. As stated
  above it’s important that the algorithm can get the minimal of these based
  on the heuristic and the cost of the path taken so far. This is why you should
  really use a good [priority queue/heap](https://en.wikipedia.org/wiki/Priority_queue)
  here. For this example I just missused a F#-Map structure which is not optimal
  (it has no O(1) access to the min. element) but it’s sufficient 😉
* `gScore`: keeps track of the costs on the path taken to a node so far – the
  algorithm updates this when it finds better paths to a node as well.
* `fScore`: is the actual cost to a node plus the heuristic from that node to
  the goal. The algorithm uses these for the priority queue in `openNodes`

The algorithm is initialized by providing the needed function in this structure:

```fsharp
type Config<'node when 'node : comparison> =
    {
        heuristic : 'node -> Score
        neighbours : 'node -> 'node seq
        distance : 'node -> 'node -> Score
        isGoal : 'node -> bool
    }
```
    
You can find the complete implementation [on github](https://github.com/CarstenKoenig/AstarDemo/blob/master/Astar/AStarAlgorithm.fs)
and it’s a bit boring and really just the direct translation from the
[pseudocode on Wikipedia](https://en.wikipedia.org/wiki/A*_search_algorithm#Pseudocode).

## Example Day13

So the complete example for day 13 works like this:

```fsharp
module Maze =
 
    let private bits n =
        let gen n =
            if n <= 0 then None else
            Some (n % 2, n / 2)
        Seq.unfold gen n
 
    let private oneBits n =
        bits n
        |> Seq.filter ( ( <> ) 0)
        |> Seq.length
 
    let isWall favNumber (x,y) =
        let nr = x*x + 3*x + 2*x*y + y + y*y + favNumber
        oneBits nr % 2 = 1
 
    let private dist (x,y) (x',y') =
        abs (x'-x) + abs (y'-y)
     
    let private validCoord (x,y) =
        x >= 0 && y >= 0
 
    let private neighbours favNumber (x,y) =
        [ (x-1,y); (x,y-1); (x+1,y); (x,y+1) ]
        |> Seq.filter 
               (fun pos -> validCoord pos &&
                           not (isWall favNumber pos))
 
    let findPath favNumber fromPos toPos =
        let config : Algorithm.Config<_> =
            {
                heuristic = fun pos -> dist pos toPos
                neighbours = neighbours favNumber
                distance = fun _ _ -> 1
                isGoal = fun pos -> pos = toPos
            }
        in config |> Algorithm.aStar fromPos
```
        
If you want to have some fun take this algorithm/file and see if you can tackle
[day 11](http://adventofcode.com/2016/day/11) with it

*hint*: you should override the comparision/equals for your node to take
advantage of the strucutre of the problem – a simple heursitic is to count
everyhting on floor 4 as 0, everything on floor 3 as 1, everything on floor 2 as
2 and floor 3 as 3.

----

**Happy Christmas everyone**


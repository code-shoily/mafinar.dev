+++
title = "Many Ways to Reach a Floor"
date = "2021-07-14T09:09:23-04:00"
author = "mafinar"
authorTwitter = "mafinar"
cover = ""
tags = ["fsharp", "100-days-of-fsharp", "advent-of-code"]
keywords = ["fsharp", "100-days-of-fsharp", "advent-of-code"]
description = "On my day 1 of #100DaysOfFSharp I solved Advent of Code 2015/1 and I am sharing the experience here"
showFullContent = false
draft = false
+++
This is day 1 of my #100DaysOfFSharp and in this post I will talk about what I learned today. I tried solving [Advent of Code 2015's Day 1](https://adventofcode.com/2015/day/1) - an easy problem that packed a lot of F# education for me.

The problem is stated in the link but the `tl;dr` of it is climbing up and down based on a specific instruction - `(` means going up while `)` means down. And the input is a lot parenthesis- reminiscent of Lisp jokes. The title of the problem: `Not Quite Lisp`

We have two parts of the puzzle here:

1. If one starts from ground floor and follows the given instruction set, the program should figure out what floor will be reached after all up/down instructions are followed.
2. If one starts from ground floor and scans the given instruction set, the program should figure out how many steps will be required to reach the basement.

0 = Ground Floor. -1 = Basement.

## Part 1

F# is a "functional first" language, which means I should be able to do things without thinking too functionally. This should also be an opportunity for me to see how to deal with mutations here, as it is more than likely that I might not be doing too many mutables in future. So let's think this problem through the procedural lens. I read up on [`mutable`](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/values/) and [`for`](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/loops-for-in-expression) in F# and came up with the following snippet. Loved that `<-` for mutation.

```fsharp
let toFloorIter instructions =
    let mutable destinationFloor = 0

    for i in instructions do
        if i = '(' then
            destinationFloor <- destinationFloor + 1
        else
            destinationFloor <- destinationFloor - 1

    destinationFloor
```

That worked well! now let's get to the "Functional" bit. Why not replace all the `(`s with `1`s and `)`s with `-1`s and sum it up? That looks like a thing that is simple and should work? Let' grab a module that gives me maps, reduces, filter etc and do it!

```fsharp
let toFloorWithMap (instructions: string) =
    instructions
    |> Seq.map(fun instruction -> if instruction = '(' then 1 else -1)
    |> Seq.sum
```

Pipes are awesome. 

Now, let's try reduction! I mean, I could (and did) totally reinvent the wheel and replace `Seq.sum` with `Seq.reduce (+)` but reducing flow could potentially save me a step in the pipeline. In other words, I can start with 0, and a reducer function that takes it, and an char from the instruction and either increments or decrements it, then by the time I am done scanning the list, I should be reaching the desired floor. In the process, I can extract the `if ... then` logic into its own function.

```fsharp
let followInstruction current instruction =
    if instruction == '(' then
        current + 1
    else
        current - 1 

let toFloorWithFold (instructions: string) =
    Seq.fold followInstruction 0 instructions
```

I said reduce but used fold, that is because I needed an initial state and the result type is not type the sequence is of. I would thank IDE and strong typing for teaching me this without having to StackOverflow (I use Jetbrains Rider). I **did** try `reduce`. The typespec is so beautifully legible:

```fsharp
List.fold : ('State -> 'T -> 'State) -> 'State -> 'T list -> 'State
List.reduce : ('T -> 'T -> 'T) -> 'T list -> 'T
```

Before going point-free there was one thing I wished to check for- I used `if then else` without checking for `(`, assuming the input is comprised solely of `(` and `)` (it actually is), however, while depending on how I pull the inputs, there could be an `"\n"` or an unwanted `" "` lying around. Also, this is an opportunity to use the match operator! So if I had to refactor `followInstruction` to use pattern matching:

```fsharp
let followInstructions current instruction =
    match instruction with
    | '(' -> current + 1
    | ')' -> current - 1
    | _ -> 0
```

...or I could `failwith` a message, but let's assume anything other than `(|)` is inconsequential.

Now let's see if we can go point free. `toFloorWithFold` can be `fun instruction -> ...` and since `f(x, y)` is `f(x)(y)`, by leaving out `instructions` parameter, I can have this return a function that takes that missing `instructions` as parameter. Which leads me to:

```fsharp
let toFloorWithFoldPointFree : seq<char> -> int = 
    Seq.fold followInstructions 0
```

And F# forced me to add typed in this version! Which led me to [this nice article](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/generics/automatic-generalization). Let's liberate the mapped version from shackles of a point:

```
let toFloorWithMapPointFree : seq<char> -> int =
    Seq.map (followInstructions 0) >> Seq.sum
```

So `>>` composes two functions, kindof like a curried `|>`. There's also a `<<`. I have never use such a construct before though I see myself using it a lot. That one-liner read as "Create a function that maps each element of the "seqable" it receives to `followInstruction` function, giving it `0`; then it just sums it".

Wow, quite a few ways to reach a floor! Now let's see how many steps we'd need to get to the basement!

## Part 2

This too is an easy problem to solve. Instead of looping over the instructions and updating the `currentFloor` TILL THE END, we stop iterating when the `currentFloor` is `-1` and the `counter` at that time is our answer. So a C style `for loop` with a `break` would be perfect for it. Or `while`, like below: 

```fsharp
let toBasementWhile (instructions: string) =
    let mutable steps = 0
    let mutable current = 0
    let instructionArray = instructions.ToCharArray()
    while current <> -1 do
        if instructionArray.[steps] = '(' then
            current <- current + 1
        else
            current <- current - 1
        steps <- steps + 1
    steps
```

I could not find a sane looking `break` policy for a loop in F#, and `ToCharArray` is a method my recent encounters with C# familiarized me with, so grabbed it. And again, it worked!

Enough with the iterative approach, let's try some good old fashioned recursion. 

```fsharp
let rec doStepsToBasement =
    function
    | (step, -1, _) -> step
    | (step, current, '(' :: instructions) -> doStepsToBasement (step + 1, current + 1, instructions)
    | (step, current, ')' :: instructions) -> doStepsToBasement (step + 1, current - 1, instructions)
    | _ -> failwith "Invalid input"

let stepsToBasementRecur (instructions: string) =
    doStepsToBasement (0, 0, instructions |> Seq.map (fun l -> l) |> Seq.toList)
```

Rest style pattern matching is one of my favorite super powers. I could just visualize an instruction as a dispenser, dispensing one instruction at a time, until it's empty, and everytime the spitted out parenthesis, the `-1`ed instruction list and all computed states are pushed back into the same function! Pretty. (That made absolutely no sense now did it?)

I searched for a `reduce_while` in `Seq` but could not find any, but there is a `scan`, which is like a dispenser of reduced items on the fly, so, why not do something like:

```fsharp
let stepsToBasement : seq<char> -> int =
    Seq.map (followInstructions 0)
    >> Seq.scan (+) 0
    >> Seq.findIndex ((=) -1)
```

And this time point-freedom came naturally to me! Didn't miss `reduce-while` much. And also, I did think about `Seq.indexed` and then scanning but luckily I found out about `findIndex` early on.

And that ends my day 1 of the 100 days of F#.

## My approach at learning things today

So this was a really interesting experience for me. Usually I learn languages (ones I learn for fun) from books- I buy a book and work through the examples, the most recent one being "C# 9.0 Pocket Reference". However in this case, I just went head-on with problem solving armed only with common-sense. I picked a problem, thought of ways to solve it, and googled generic applicable terms to come up with F# artifacts to help me construct outcomes. This took longer time and slower (initial) progress but it did give me that feeling of "struggle" and "resistance" which leads to true appreciation of the programming language. I had no idea what an F# `Seq` meant, what separated `reduce` from `fold`, or how I would mutate the mutable. Those would probably not be colocated in one chapter of (m)any books, but I dug scattered information from the web and stayed in REPL until everything made sense. And that was a fun, fulfilling learning experience, more of which I look forward too.


## Any final words?

The official F# documentation is beyond awesome. The article on Automatic Generalization had little to do with my work today but I stumbled upon it any way, and got pretty well enlightened. Almost every article I read from the docuemtation site has been enjoyable and to the point. For example, [this page](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/fsharp-collection-types) is something every language should cover! As for the day 1 in general, this has been a much entertaining day for me. I can safely say that I am looking forward to the 99 other days and beyond! My gratitude to everyone involved in creating and nurturing this amazing language!


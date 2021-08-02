+++
title = "Stacked Reactions"
date = "2021-08-02T00:16:59-04:00"
author = "mafinar"
authorTwitter = "mafinar"
cover = ""
tags = ["fsharp", "100-days-of-fsharp", "advent-of-code"]
keywords = ["fsharp", "100-days-of-fsharp", "advent-of-code"]
description = "In this blog, I write about how I solved Advent of Code 2018/5 with F#. Part of my #100DaysOfFSharp, Day 17"
showFullContent = false
+++

##### tl;dr I solved [Advent of Code Year 2018](//adventofcode.com/2018/day/5), Day 5 and [here is the code](//github.com/code-shoily/AdventOfCode/blob/master/fsharp/Y18/Year2018Day05.fs). 

When I first encountered LIFO (i.e. Stack) data structures back in my Algorithmics course, I found the implementation to be easy compared to, say linked lists, however, to identify its use in a real life problem (for instance, parenthesis matching) was something I had struggled with. I mean, I would probably be able to solve one with stiching expressions and statements but to immediately realize that the problem is addressable with a stack never came to me easily. I eventually overcame that (Or so I think), and today's post, and the advent of code puzzle involved, involved a LIFO.

So what does the description say? To summarize, any `<lower-case><upper-case>` or `<upper-case><lower-case>` sequence of the same letter would need to evaporate in a sequence of letters (unit type, they say), and you keep collapsing such pair until there remains no other. That's when you output the length of a non-reducable (reactible, they say) sequence of units.

To give an example (I copied the example from the site as-is):

```
dabAcCaCBAcCcaDA  The first 'cC' is removed.
dabAaCBAcCcaDA    This creates 'Aa', which is removed.
dabCBAcCcaDA      Either 'cC' or 'Cc' are removed (the result is the same).
dabCBAcaDA        No further actions can be taken.
```

The result of the above sequence would be the length of the non-reducible sequence `dabCBAcaDA`, or `10`. Now they slapped me with a `50_000` character sequence that would follow the process above.

When writing programs in F#, and I only had 17 days of it, I found out that treating input as a sequence, and pouring that sequence through a series of transformation helps me produce tolerable looking code. And to assume that everything already happened (as oppose to my function making them happen) helps more.

For instance, let's see how I would see the sequence of `units` as mentioned here transform till I get my desired result:

```
daBbAcCd                                                                    The input.
'd' 'a' 'B' 'b' 'A' 'c' 'C' 'd'                                             As Character Stream
('d' 'a') ('a' 'B') ('B' 'b') ('b' 'A') ('A' 'c') ('c' 'C') ('C' 'd')       In two-s
('d' 'a') ('a' _) (_ 'A') ('A' _) (_ 'd')                                   Get rid of (a, b) where (lower a) = (lower b)
'd' 'a' 'A' 'd'                                                             Stringify them
('d' 'a') ('a' 'A') ('A' 'd')                                               In two-s (Again)
('d' _) (_ 'd')                                                             Get rid of (a, b) where (lower a) = (lower b)  
'd' 'd'
```

See, that transformation works! In paper and pencil, and given `Seq.windowed` exists and there is nothing one can't do with `Seq.fold`, I am fairly certain that I can whip up a nice little F# code out of it. Succint? Yes. Elegant? I am not quite sure.

So, I gave myself fifteen more minutes to think a bit more on this. I used the term "pour" earlier, so what if I pour in the `unit`-s, hold what I just poured and match it with the next drop? Let's sketch it again:

```
daBbAcCd                            The input.
'd' 'a' 'B' 'b' 'A' 'c' 'C' 'd'     As Character Stream
'd' | 'a' 'B' 'b' 'A' 'c' 'C' 'd'   Partition after head, 'd' | 'a', move right.
'd' 'a' | 'B' 'b' 'A' 'c' 'C' 'd'   'a' | 'B', move right
'd' 'a' 'B' | 'b' 'A' 'c' 'C' 'd'   'B' | 'b', delete both sides of partition, stay
'd' 'a' | 'A' 'c' 'C' 'd'           'a' | 'A', delete both sides of partition, stay
'd' | 'c' 'C' 'd'                   'd' | 'C', move right
'd' 'c' | 'C' 'd'                   'c' | 'C', delete both sides of partition, stay
'd' | 'd'                           'd' | 'd', move right
'd' 'd' |                           DONE
```

Now this makes some sense, I can focus on line 1, and go down at line 10 in one breath, without ever backtracking my thoughts. I will now implement this in code.

Let's look into what's on the left and right sides of the `|`. If I take one character from the left of the sequence, and compare it to what is at the right most of the left partition and use a comparison function to decide whether it will `append` to the left partition, or go down and take that right-most part with it, that sounds like similar operation to what the sketch above shows.

In other words, at any given snapshot of `0 1 2 3 4 | 5 6 7`, `4` will be compared with `5` and if they are to react, then the data structure will look like `0 1 2 3 | 6 7` and if they are to not react, then they would look like `0 1 2 3 4 5 | 6 7`. So, the left data structure needs only the append or remove fast to the right most element (aka the last inserted item). So before I (finally), get into the code, I'd need the following:

1. A mechanism to read one element of a sequence
2. A function that lets me compare two characters and decide whether they react or not, and 
3. A stack, that will maintain the `visited` values and with the exception of the top-most (right-most) character, it is guaranteed that no reaction can be possible.

Here's how my `react` function looks like:

```fsharp
let react sequence =
    sequence
    |> Seq.fold
        (fun visited unitType ->
            match visited with
            | right :: nonReactive when abs (unitType - right) = 32 -> nonReactive
            | _ -> unitType :: visited)
        List<int>.Empty
    |> Seq.length
```

This worked, and gave me a sweet `*`. Oh, I decided to `int` my character stream, though I realize I could just as well kept it as a string sequence and compared with converting both to either lower or upper case. 

Also, since I am using `fold` anyway, why not compute the `length` on the fly and keep it as a `state`? And get some point-freedom while at it? And let's just use some Stack terminology.

```fsharp
let react =
    Seq.fold
        (fun (length, stack) unitType ->
            match stack with
            | top :: bottom when abs (unitType - top) = 32 -> (length - 1, bottom)
            | _ -> (length + 1, unitType :: stack))
        (0, List<int>.Empty)
    >> fst
```
I am happy with this code. And that concludes part 1.

About part 2, well, it's just part 1 with some pseudo-brute-forcing. What if we removed all `a|A`-s from the sequence and ran the reaction? What if we removed `b|B`s? `c|C`-s? So you can see, we just need to call `react` function on 26 different inputs, and find the shortest non-reducable sequence. And this code looked like:

```fsharp
let solvePart2 input =
    seq { 65 .. 90 }
    |> Seq.map
        (fun currentUnit ->
            input
            |> parse
            |> Seq.filter
                (fun unitType ->
                    unitType <> currentUnit
                    && unitType <> (currentUnit + 32))
            |> react)
    |> Seq.min
    |> output
```

So that concludes my day 17. I know my last blog post was labelled `Day 1` and this one `Day 17` (Or is it 16?), I solved 15 other problems in between while learning the language and didn't quite get the time to write about it. This one however, I wrote as I solved the problem and that was fun. Maybe I will do it more often.

Until I solve another problem, or pick up one of the solve problems to discuss here, have a great evening!

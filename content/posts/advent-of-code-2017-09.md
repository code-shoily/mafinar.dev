+++
title = "Picking garbages in groups"
date = "2021-11-02T00:34:12-04:00"
author = "Mafinar K."
authorTwitter = "@mafinar"
cover = ""
tags = ["fsharp", "100-days-of-fsharp", "advent-of-code", "PragProWriMo"]
keywords = ["fsharp", "100-days-of-fsharp", "advent-of-code", "prag-pro-wri-mo"]
description = "In this post, I solve Advent of Code 2017, Day 9 puzzle and explain how I did it in F#"
showFullContent = false
+++
In today's post, I will describe my solution to Advent of Code year 2017, day 9's challenge. Let's [read the challenge description](https://adventofcode.com/2017/day/9) before the explanation.

## Part 1

Part 1 of the problem expects us to report the total number of groups, as collected from the stream of characters. The rules are simple:

* `{` marks the beginning of the group **unless that's found in the garbage**
* `}` marks the end of the group **unless that's found in the garbage**
* First group encounter will have the level of `1`, nested groups will have their levels as their depth, as an example, the innermost group of `{{{}}}` will have level of `3`.
* Groups are separated by commas.

Now that is the easy part, let's get to the garbage.

* `<` marks the beginning of garbage, anything after that are garbage (including `{` or `}`), until
* `>` is found. That marks the end of the garbage.
* If within the garbage, an `!` is encountered, then the following character is discarded, whether it is another `!` or any other delimiters.

The puzzle description has great examples of what constitutes garbages and groups, and streams that would be puzzling to the eyes (such as `}` inside garbage or `>` after discards) are also there, so this problem is very test-case friendly.

Our goal for this part is to add up all the group levels accummulated from the stream.

### The solution

This problem smelled like state machine and when dealing with F#, I could immediately picture a solution in my head. But what would aid me more is a picture of the state transitions. So I fired up a [Mermaid Live](https://mermaid.live/) and type in the following state transitions:

```
stateDiagram-v2
    [*] --> GroupStarted: "{" level++
    GroupStarted --> GroupEnded: "}" level--
    GroupStarted --> GarbageStarted: "<"
    GarbageStarted --> GarbageEnded: ">"
    GarbageEnded --> GroupEnded: "}"
    GarbageStarted --> DiscardNext: "!"
    DiscardNext --> GarbageStarted: Any character
    GarbageStarted --> GarbageStarted: Any character EXCEPT ">" and "!"
    GroupEnded --> GroupEnded: ","
    GroupEnded --> GroupStarted: "{"
    GroupEnded --> [*]: All characters processed
```

And the following image is produced:

![Alt text](/state-chart.png "State Machine")

We can see that, there is a `GroupMode` and a `GarbageMode` and the upcoming character has different effect to the overall state in different `Mode`-s. There is also a `DiscardMode` that discards the upcoming character, thereby affecting the closing of `GarbageMode`.

How would the F# look like?

Since we are dealing with streams, we can go for a recursive function, that processes a list in `head :: rest` pattern. And we can have a state type representing the state of the state machine. We can then have a `collection` function that takes in a state, and then recursively calls itself, processing one character at a time - a character that resides inside the state.

Let's see what can we have in the state type-

```fsharp
type CollectionState =
    { Stream: char list
      Count: int
      GarbageMode: bool
      DiscardMode: bool
      Total: int }
```

So, we have a `GarbageMode` and a `DiscardMode`, `Stream` representing the characters we have left to process (and classify as a garbage or collect as a group), and `Total` which is a sum-total of the groups collected. The `Count` variable here adds the depth of the group.

So from that definition, we have the terminating condition of the processing, when the `Stream` is empty (aka we have processed everything). If we patternize the `Stream` as `current :: upcoming`, then just by giving `Stream` the value of `upcoming`, we are progressing through the list, churning characters. Also, we want the `Total` to have the total number of groups.

So, let's see how a preliminary version of the function look like:

```fsharp
let rec collect_groups (state: CollectionState) =
    match state with
    | { Stream = []; Total = total } -> total
    | { Stream = _ :: remaining
        Total = total } ->
        collect_groups
            { state with
                    Stream = remaining
                    Total = total + 1 }
```

Now, if we pass `{ Stream = input; Count = 1; GarbageMode = false; DiscardMode = false; Total = 0 }` to it where `input` is a string `|> List.ofSeq`ified, we will, unsurprisingly, get the number of characters in that string. Because we didn't deal with the `Mode`-s, or character effects as portrayed in the state-chart above.

Let's now assume that there is just one mode - the group mode, and our input is a lot simpler one - one comprising of `{}` and `,` to separated them. Let's refer back to the state chart and see what we can do - 

```fsharp
let rec collect_groups (state: CollectionState) =
    match state with
    | { Stream = []; Total = total } -> total
    | { Stream = '{' :: remaining
        Count = count
        Total = total } ->
        collect_groups
            { state with
                    Stream = remaining
                    Count = count + 1
                    Total = total + count }
    | { Stream = '}' :: remaining
        Count = count } ->
        collect_groups
            { state with
                    Stream = remaining
                    Count = count - 1 }
    | { Stream = ',' :: remaining } -> collect_groups { state with Stream = remaining }
    | _ -> failwith "Invalid State"
```

This time, we compute the level and sum them up, provided we are given only `{`, `}` amd `,`. For example, giving `"{{{}},{},{},{{}}}" |> List.ofSeq` would give us `15`. However, the input given in the problem wouldn't, because I intentionally made invalid state irrepresentable through that `failwith`.

So when would a `<` constitute the beginning of a garbage? When `GarbageMode` is `false` and `DiscardMode` is `false`. And `>` will mark the end of a garbage when `GarbageMode` is `true` and `DiscardMode` is `false`. So, the pattern above will be updated with a `{GarbageMode = false}` to ensure that group is formed if and only if it is not in `GarbageMode`.

Now, if we refer to the state chart, we see that `DiscardNext` only takes place AFTER `GarbageStarted` happens. In other words, there is no scenario where `DiscardNext` is `true` and `GarbageMode` is `false`, that would be impossible state, therefore, our matching should reflect that. And when in `DiscardMode`, the flow is just `{Stream = rest}`ed, and the `DiscardMode` is immediately turned `off`.

Now, if we combine the pattern above with the state chart, we can easily visualize and form the function in full - 

```fsharp
let rec collect_groups (state: CollectionState) =
    match state with
    | { Stream = []; Total = total } -> total
    | { Stream = '{' :: rest
        Count = count
        GarbageMode = false
        Total = total } ->
        { state with
                Stream = rest
                Count = count + 1
                Total = count + total }
        |> collect_groups
    | { Stream = '}' :: rest
        Count = count
        GarbageMode = false } ->
        { state with
                Stream = rest
                Count = count - 1 }
        |> collect_groups
    | { Stream = '<' :: rest
        GarbageMode = false } ->
        { state with
                Stream = rest
                GarbageMode = true }
        |> collect_groups
    | { Stream = _ :: rest
        DiscardMode = true } ->
        { state with
                Stream = rest
                DiscardMode = false }
        |> collect_groups
    | { Stream = '!' :: rest
        GarbageMode = true } ->
        { state with
                Stream = rest
                DiscardMode = true }
        |> collect_groups
    | { Stream = '>' :: rest
        GarbageMode = true } ->
        { state with
                Stream = rest
                GarbageMode = false }
        |> collect_groups
    | { Stream = _ :: rest } -> { state with Stream = rest } |> collect_groups
```
And it worked on my machine!

A few things I learned while doing this:

* Learned to love `{ state with ...}` instead of forming the whole thing
* Made sure to not provide unnecessary patterns (i.e. `GarbageMode = true` when that branch can never have `GarbageMode` as true), this took a refactor and I think wasn't totally necessary.
* Resisted the temptation to use tuples. Tuple could have been fine, but I preferred having the attributes in my face, instead of just a group of values.

This problem was fun to solve, and I thought I'd write about it here. Also, this had similarities with the [first blog post I wrote in this site](/posts/many-ways-to-reach-a-floor/) - with an added complication.

Now, about Part 2 of the problem, it was very much similar to part 1 with different logic to compute the `Count` and `Total`. And the fact I called `Count` `Count` instead of the more semantically appropriate `Level` has to do with the fact I reused this state for `Part 2` and extracted the snippets from my original code. Both solutions can be [viewed from the repository](https://github.com/code-shoily/AdventOfCode/blob/master/fsharp/Y17/Year2017Day09.fs). 




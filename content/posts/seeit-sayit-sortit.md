---
title: See it, Say it, Sort it!
subtitle: An example of how to contribute to the F# compiler as a beginner
date: 2022-12-09
author: "Edgar"
tags: [fsharp, compiler, warning]
draft: false
---

> An F# [Advent Calendar in English 2022](https://sergeytihon.com/2022/10/28/f-advent-calendar-in-english-2022/) blog post. Big thanks to [Sergey Tihon](https://twitter.com/sergey_tihon) for organising F# weekly

If you live in the UK and use public transport you might have heard the [See it, Say it, Sorted](https://www.btp.police.uk/police-forces/british-transport-police/areas/campaigns/see-it-say-it-sorted/) slogan encouraging you to help in case you see something that does not feel right by saying it, and then it will be sorted(not by you but by the police).
So in this post, I wanted to play with this phrase and encourage you to do the same but for F#.

## Introduction
In this post, I will show you that contributing to the F# compiler is not difficult, You might be thinking you need to be an expert and know a lot of low-level concepts, while this is true if you want to work on a specific feature, the reality is that there are multiple areas where you can start without previous experience ie [good first issues](https://github.com/dotnet/fsharp/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22+label%3A%22help+wanted%22):
- Fixing typos
- IDE's quick fixes
- Simply adding more testing to existing features
- Improving error reporting ie: new compiler error or warning where there is none

In this post, I will focus on the last point, [adding a new warning](https://github.com/dotnet/fsharp/pulls?q=is%3Apr+author%3Aedgarfgp+is%3Aclosed).
I will show you how to add it and test it.

By doing this you will learn:

- More about F# and get familiar with concepts that you might not have seen before
- know the F# compiler better and how it works.

## Problem

If you use F# [union types](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/discriminated-unions) with [pattern matching](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/pattern-matching) you seen the following code:

```fsharp
type HelpBy =
   | SeeIt of bool
   | SayIt
   | SortIt

let contribute msg =
    match msg with
    | SeeIt _ -> "Something that needs to be fixed" 
    | SayIt _ -> "Raise an issue on GitHub" // No compiler warning
    | SortIt _ -> "if you can, submit a pull request" // No compiler warning
```

> This had me confused when I was learning F# and I thought might be a bug because I was expecting a compiler warning for the unused variable `_` in the `SayIt` and `SortIt` cases.
Because both cases do not hold a value, so the `_` is not needed.

So I followed my own advice and went to the F# compiler [repo](https://github.com/dotnet/fsharp/issues) to raise an issue `(Say it)`, fortunately, it was already [raised](https://github.com/dotnet/fsharp/issues/13851).
So I decided to give it a try `(Sort it)`.

## Analysis
We can start by choosing a minimal reproducible example, so we can test our changes, and focus on the problem.
In our case, a single case union type with a wildcard pattern match will be a good starting point.

```fsharp
type HelpBy =
    |  SayIt
 
match msg with
| SayIt _ -> "Raise an issue on GitHub"
```
Then, we need to understand how our source code is represented in the compiler. For that, we need to learn our first concept `AST` aka [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree).

> **Abstract Syntax Tree** is a way of representing the structure of a program in a hierarchical tree-like format. It is used to make it easier for a computer to understand and process a program. Each node in the tree represents a different part of the program, such as a variable, function, or statement. This allows the computer to quickly and accurately execute the instructions in the program. This will be important for us in the implementation part

F# ecosystem has a source code formatter called [Fantomas](https://fsprojects.github.io/fantomas/) which includes an AST visualizer called [Fantomas tool](https://fsprojects.github.io/fantomas-tools/).

![AST Representation](/images/ast-representation.png)

From the AST representation we can highlight the following:
- `SynMatchClause` is the root node and tells us that we are dealing with a pattern match
- `LongIdent` is the pattern match, in our case, it is a single case union type
- `SynLongIdent` is the union case, in our case, it is `SayIt`
- `Pats` is the wildcard pattern, in our case, it is `_`
- `Const` is the string literal, in our case, it is `"Raise an issue on GitHub"`

If we look at the compiler `SynPat` type we can see that the `SynPat` type includes our `LongIdent`. This means that we can use the `LongIdent` to get the `SynLongIdent` and then we can check if the `SynArgPats` contains a wildcard pattern.

```fsharp
[<NoEquality; NoComparison; RequireQualifiedAccess>]
type SynPat =
    | Const of constant: SynConst * range: range
    | Wild of range: range
    | As of lhsPat: SynPat * rhsPat: SynPat * range: range
    | LongIdent of
        longDotId: SynLongIdent *
        extraId: Ident option *  // holds additional ident for tooling
        typarDecls: SynValTyparDecls option *  // usually None: temporary used to parse "f<'a> x = x"
        argPats: SynArgPats *
        accessibility: SynAccess option *
        range: range
```

Now that we know how the code is represented in the compiler. We can start by adding a unit test to validate the existing behavior. This will help us to validate our changes before and after.

```fsharp
 [<Fact>]
 let ``Pattern discard allowed for union case that takes no data`` () =
      FSharp """
module Tests =
    type HelpBy = SayIt
    let msg= SayIt
    match msg with
    | SayIt _ -> "Raise an issue on GitHub"
    """ 
    |> typecheck
    |> shouldSuceed
```
> Adding a unit test is really easy. Just use the  `FSharp """ Your Code """` function and then call `typecheck` and `shouldSuceed` functions.

To learn more about how the compiler is structured, I went on reading the [documentation](https://github.com/dotnet/fsharp/blob/main/docs/index.md), it is easy to read, but also extensive! You can find explanations and videos about almost everything inside it.

After reading it I found that a good starting point might be [CheckPatterns.fs](https://github.com/dotnet/fsharp/blob/55c17665cb944a9f4580c6b945d190410d1d0989/src/Compiler/Checking/CheckPatterns.fs#L598) where the compiler check a long identifier(described in the AST as `LongIdent`) case that has been resolved to a union case or F# exception constructor. So we know we are in the correct place.

## Implementation

But before we start modifying the code, we need to find a clear and meaningful warning message. So I went to the related issue to discuss this. I got a lot of feedback and suggestions, and I ended up with `Pattern discard is not allowed for union case that takes no data`.
Now we need to add our error message to [FSComp.txt](https://github.com/dotnet/fsharp/blob/55c17665cb944a9f4580c6b945d190410d1d0989/src/Compiler/FSComp.txt#L1661). This is the file that contains most of the error messages that the compiler uses.

We need to provide a unique error code, name and message.
> 3548, matchNotAllowedForUnionCaseWithNoData, Pattern discard is not allowed for union case that takes no data.

If you follow the logic of how patterns are checked you will get a function called [TcPatLongIdentUnionCaseOrExnCase](https://github.com/dotnet/fsharp/blob/55c17665cb944a9f4580c6b945d190410d1d0989/src/Compiler/Checking/CheckPatterns.fs) and in it, you will find the following code:

```fsharp
let args, extraPatternsFromNames =
    match args with
    | SynArgPats.Pats args -> args, []
```
> We can see in this function, it is not checking for wildcards or named patterns, so we need to add it.

One way to fix it would be to add new checks to the pattern matching for the wildcard or named patterns as follows:
```fsharp
let args, extraPatternsFromNames =
     match args with
     | SynArgPats.Pats args ->
         match args with
         | [ SynPat.Wild _ ] | [ SynPat.Named _ ] when argNames.IsEmpty  ->
             warning(Error(FSComp.SR.matchNotAllowedForUnionCaseWithNoData(), m))
```
> Now this function checks for `SynPat.Wild` and `SynPat.Named` patterns in `SynArgPats.Pats` and raises a warning if the union case has no data as `argNames` is empty.

Now we have a code that makes sense from the implementation point. We need to validate this change and the best way to do it is by adding as many tests as you can. In our case, we can add those tests to [UnionCasePatternMatchingErrors.fs](https://github.com/dotnet/fsharp/blob/main/tests/FSharp.Compiler.ComponentTests/ErrorMessages/UnionCasePatternMatchingErrors.fs) in the `FSharp.Compiler.ComponentTests` project.
We need to make sure our warning is working on single and multi-case union types. For instance, we now expect a warning in our initial sample.

```fsharp
type HelpBy =
    | SeeIt of bool
    | SayIt
    | SortIt

let contribute msg =
    match msg with
    | SeeIt _ -> "Something that needs to be fixed" //No warning
    | SayIt _ -> "Raise an issue on GitHub" //warning: FS3548 Pattern discard is not allowed for union case that takes no data
    | SortIt _ -> "if you can, submit a pull request" //warning: FS3548 Pattern discard is not allowed for union case that takes no data

```

Now our previous test will fail because we are now raising a warning `FS3548` :

```fsharp
 [<Fact>]
 let ``Pattern discard allowed for union case that takes no data`` () =
      FSharp """
module Tests =
    type HelpBy = SayIt
    let msg= SayIt
    match msg with
    | SayIt _ -> "Raise an issue on GitHub"
    """ 
    |> typecheck
    |> shouldFail
    |> withSingleDiagnostic (Warning 3548, Line 6, Col 12, Line 7, Col 13, "Pattern discard is not allowed for union case that takes no data.")
```

Now it is time to raise a [Pull request](https://github.com/dotnet/fsharp/pull/14055) to validate our fix. The compiler team will help and guide you through the process, ask you to cover more cases, testing to make sure the implementation is bulletproof.

Congratulations! We have just contributed to the F# compiler, and it was not that hard, right?

## IDE support

We now have a new compiler warning that will make our life easier when working with F#. But we can do better and implement a quick fix in our preferred IDE.

We will use Jetbrains [Rider](https://www.jetbrains.com/rider/) as our editor of choice.
This can be a whole new post about how I [implemented](https://github.com/JetBrains/resharper-fsharp/pull/444) the quick fix.

At the time of this post [resharper-fsharp](https://github.com/JetBrains/resharper-fsharp) is not using the latest compiler version so it won't report this warning.
But our quick fix will also work with some small modifications.

![Quick fix](/images/union-case-quick-fix.gif)

## Conclusion
- Can you contribute to the compiler without previous experience? YES, you can.
- Is it hard? Not really, you need to be familiar with some concepts like AST and the F# language.
- Is it worth it? YES, you will learn a lot about the compiler and the language. You will also be able to help the community and the compiler team.
- Is it fun? YES, you will be able to see your changes in action and help the community.
- Is it hard to get started? Not really, you can start by reading the [documentation](https://github.com/dotnet/fsharp/blob/main/docs/index.md) and ask questions in the Slack [channels](https://fsharp.org/guides/slack/)
- Is it hard to get your changes merged? Not really, the compiler team is very helpful and will guide you through the process.

---
title: Adding a F# compiler warning
date: 2022-09-12
author: "Edgar"
tags: [fsharp, compiler, warning]
draft: false
---

> An F# [Advent Calendar in English 2022](https://sergeytihon.com/2022/10/28/f-advent-calendar-in-english-2022/) blog post. Big thanks to [Sergey Tihon](https://twitter.com/sergey_tihon) for organising F# weekly

If you live in the Uk and use public transport you might have heard a similar [slogan](https://www.btp.police.uk/police-forces/british-transport-police/areas/campaigns/see-it-say-it-sorted/) encouraging you to help in case you see something that does not feel right by saying it and then it will be sorted(not by you but by the police). So in this post, I wanted to play with this phrase and encourage you to do the same but for F#.

## Introduction
I will explain what you need to know in order to add a warning to the F# compiler. Based on my experience [contributing](https://github.com/dotnet/fsharp/pulls?q=is%3Apr+author%3Aedgarfgp+is%3Aclosed)

You might be thinking you need to be an expert and know a lot of low-level concepts, while this is true if you want work on a specific features, the reality is that there are multiple areas where you can start without previous experience:
 - Fixing typos
 - Improving error reporting (by suggesting a more clear error message)
 - IDE's quick fixes
 - Simply adding more testing to existing features

By doing this you will learn more about F# and get familiar with concepts that you might not have seen before. You will also get to know the F# compiler better and how it works. Yo can see the compiler as a new domain that you can explore and learn more about.

I’m going to use one of my contributions to show and hopefully explain what are the minimum concepts that you needed to know.

## Problem

If you use F# [union types](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/discriminated-unions) and [pattern matching](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/pattern-matching) you have seen the following behaviour:

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
> No compiler warning when using a wildcard aka underscore on field cases without data ie `SayIt` or `SortIt`

This had me confused when I was learning F# and I thought might a bug `(See it)`. So I followed my own advice and went to the F# compiler [repo](https://github.com/dotnet/fsharp/issues) to raise an issue `(Say it)`, fortunately, it was already [raised](https://github.com/dotnet/fsharp/issues/13851).
So I decided to give it a try `(Sort it)`.

## Analysis 

First, we can star by choosing a minimal repro. In our case a single case union type with a wildcard pattern match will be a good staring point.

```fsharp
type HelpBy =
    |  SayIt
 
match msg with
| SayIt _ -> "Raise an issue on GitHub"
```
Then, we need understand how our source code is represented in the compiler. For that we need to learn our first concept `AST` aka [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree).

> **Abstract Syntax Tree** is a way of representing the structure of a program in a hierarchical tree-like format. It is used to make it easier for a computer to understand and process a program. Each node in the tree represents a different part of the program, such as a variable, function, or statement. This allows the computer to quickly and accurately execute the instructions in the program. This will be important for us in the implementation part

F# has an extraordinary tool called [fantomas](https://fsprojects.github.io/fantomas-tools/) that can help us to visualise the AST.

Our minimal repro will be represented as follows:

```fsharp
SynMatchClause
    (LongIdent
        (SynLongIdent ([SayIt], [], [None]), None, None,
            Pats [Wild], None),
            None,
            Const
            (String ("Raise an issue on GitHub", Regular)))
```
From the AST representation we can highlight the following:
 - `SynMatchClause` is the root node and tells us that we are dealing with a pattern match
 - `LongIdent` is the pattern match, in our case, it is a single case union type
 - `SynLongIdent` is the union case, in our case, it is `SayIt`
 - `Pats` is the wildcard pattern, in our case, it is `_`
 - `Const` is the string literal, in our case, it is `"Raise an issue on GitHub"`

Next step, if we can look at the compiler `SynPat` type we can see that the `SynPat` type includes our `LongIdent`. This means that we can use the `LongIdent` to get the `SynLongIdent` and then we can check if the `SynArgPats` contains a wildcard pattern.

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

Now that we know how the code is represented in the compiler. We can start by adding a test to validate the existing behavior and from there start modifying some code.

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
After reading the extensive [documentation](https://github.com/dotnet/fsharp/blob/main/docs/index.md) about how the compiler is structured . I found that a good starting point might be:
- [CheckPatterns.fs](https://github.com/dotnet/fsharp/blob/55c17665cb944a9f4580c6b945d190410d1d0989/src/Compiler/Checking/CheckPatterns.fs#L598) where the compiler check a long identifier(described in the AST as `LongIdent`) case that has been resolved to a union case or F# exception constructor. So we know we are in correct place.

## Implementation

But before we start modifying code, we need to find a clear an meaningful waring message.

What about `Pattern discard is not allowed for union case that takes no data.` seems to be a good candidate.
We need to add our error message to [FSComp.txt](https://github.com/dotnet/fsharp/blob/55c17665cb944a9f4580c6b945d190410d1d0989/src/Compiler/FSComp.txt#L1661), so we can use in our implementation.

If you follow the logic of how patterns are checked you will get a function called [TcPatLongIdentUnionCaseOrExnCase](https://github.com/dotnet/fsharp/blob/55c17665cb944a9f4580c6b945d190410d1d0989/src/Compiler/Checking/CheckPatterns.fs) and in it you will find the following code:

```fsharp
// We can see that the compiler does not check for argument names on union types
let args, extraPatternsFromNames =
         match args with
         | SynArgPats.Pats args -> args, []
```

So we can add a check for `SynPat.Wild` and `SynPat.Named` based on if a union type has arguments names.

```fsharp
let args, extraPatternsFromNames =
     match args with
     | SynArgPats.Pats args ->
         match args with
         | [ SynPat.Wild _ ] | [ SynPat.Named _ ] when argNames.IsEmpty  ->
             warning(Error(FSComp.SR.matchNotAllowedForUnionCaseWithNoData(), m))
```

Now we have a code that make sense from the implementation point. We need to validate this change and the best way to do it is adding as many [tests](https://github.com/dotnet/fsharp/blob/main/tests/FSharp.Compiler.ComponentTests/ErrorMessages/UnionCasePatternMatchingErrors.fs) as you can. In our case we need to make sure our warning is working on single and multi case union types. For instance we now expects a waring in our initial sample.

```fsharp
type HelpBy =
    | SeeIt of bool
    | SayIt
    | SortIt

let contribute msg =
    match msg with
    | SeeIt _ -> "Something that needs to be fixed" //No warning
    | SayIt _ -> "Raise an issue on GitHub" //warning : FS3548 Pattern discard is not allowed for union case that takes no data
    | SortIt _ -> "if you can, submit a pull request" //warning : FS3548 Pattern discard is not allowed for union case that takes no data

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

Now it is time to raise a [Pull request](https://github.com/dotnet/fsharp/pull/14055) to validate our fix. The compiler team will help and guide you through the process, ask you to cover more cases, testing to make sure the implementation is bullet proof.

In summary, We now have a new compiler warning that wil make our life easier when working with F#. But we can do better and implement a quick fix in our preferred IDE.

## IDE support

For this we will use Jetbrains [Rider](https://www.jetbrains.com/rider/) as our editor of choice. This can be a whole new post about how I implemented the quick fix. At the time of this post [resharper-fsharp](https://github.com/JetBrains/resharper-fsharp) is not using the latest compiler version but our quick fix will also work with some small modifications.

![Quick fix](/images/union-case-quick-fix.gif)

## Conclusion
 - Can you contribute to the compiler without previous experience ?. YES you can.
 - Is it hard ?. Not really, you need to be familiar with some concepts like AST and the F# language.
 - Is it worth it ?. YES, you will learn a lot about the compiler and the language. You will also be able to help the community and the compiler team.
 - Is it fun ?. YES, you will be able to see your changes in action and help the community.
 - Is it hard to get started ?. Not really, you can start by reading the [documentation](https://github.com/dotnet/fsharp/blob/main/docs/index.md) and ask question in the Slack [channels](https://fsharp.org/guides/slack/)
 - Is it hard to get your changes merged ?. Not really, the compiler team is very helpful and will guide you through the process.

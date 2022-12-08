---
title: See it, Say it, Sort it!
date: 2022-12-02T11:31:23+01:00
author: 'Edgar'
tag: fsharp, compiler
draft: false
---

> An F# [Advent Calendar in English 2022](https://sergeytihon.com/2022/10/28/f-advent-calendar-in-english-2022/) blog post. Big thanks to [Sergey Tihon](https://twitter.com/sergey_tihon) for organising F# weekly

If you live in the Uk and use public transport you might have heard a similar [slogan](https://www.btp.police.uk/police-forces/british-transport-police/areas/campaigns/see-it-say-it-sorted/) encouraging you to help in case you see something that does not feel right by saying it and then it will be sorted(not by you but by the police). So in this post, I wanted to play with this phrase and encourage you to do the same but for F#.

In this post, I will explain what you need to do by sharing my experience [contributing](https://github.com/dotnet/fsharp/pulls?q=is%3Apr+author%3Aedgarfgp+is%3Aclosed) to the compiler without previous experience.

You might be thinking you need to be an expert and know a lot of low-level concepts, while this is true if you want to implement specific language features, the reality is that there are multiple areas where you can start like typos, error reporting(by suggesting a more clear error message), IDE quick fixes, or by simply adding more testing to existing features. By doing this you will learn more about F# and compilers in general.

I’m going to use one of my contributions to show and hopefully explain what are the minimum concepts that I needed to know.

## Introduction

If you use F# and used [union types](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/discriminated-unions) and [pattern matching](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/pattern-matching) you have seen the following behaviour:

```fsharp
type HelpBy =
   | SeeIt of bool
   | SayIt
   | SortIt

let contribute msg =
    // No compiler warning
    match msg with
    | SeeIt _ -> "Something that needs to be fixed"
    | SayIt _ -> "Raise an issue on GitHub" 
    | SortIt _ -> "if you can, submit a pull request"

```

This had me confused when I was learning F# and I thought was a bug(See it). So I followed my own advice and went to the F# compiler [repo](https://github.com/dotnet/fsharp/issues) to raise an issue(Say it), fortunately, it was already [raised](https://github.com/dotnet/fsharp/issues/13851). So I decided to give it a try.

## Analysis 

I started by choosing a minimal repo, in our case is:

```fsharp
type HelpBy =
    |  SayIt
 
match msg with
| SayIt _ -> ()
```

And looks at its [AST(Abstract Syntax Tree)](https://en.wikipedia.org/wiki/Abstract_syntax_tree) representation. F# has an extraordinary tool called [fantomas](https://fsprojects.github.io/fantomas-tools/)

```fsharp
SynMatchClause
    (LongIdent
        (SynLongIdent ([SayIt], [], [None]), None, None,
            Pats [Wild], None),
            None,
            Const
            (String ("Raise an issue on GitHub", Regular)))
```


**Abstract Syntax Tree** is a way of representing the structure of a program in a hierarchical tree-like format. It is used to make it easier for a computer to understand and process a program. Each node in the tree represents a different part of the program, such as a variable, function, or statement. This allows the computer to quickly and accurately execute the instructions in the program. This will be important for us in the implementation part
 
Now that we know how the code is represented in the compiler. We can start by adding a test to validate the existing behavior and from there start modifying some code.

```fsharp
 [<Fact>]
 let ``Pattern discard allowed for union case that takes no data`` () =
      FSharp """
module Tests
type HelpBy = SayIt
let msg= SayIt
match msg with
| SayIt _ -> ()
    """ 
     |> typecheck
     |> shouldSuceed
```

After reading the extensive [documentation](https://github.com/dotnet/fsharp/blob/main/docs/index.md) about how the compiler is structured . I found that a good starting point might be [CheckPatterns.fs](https://github.com/dotnet/fsharp/blob/55c17665cb944a9f4580c6b945d190410d1d0989/src/Compiler/Checking/CheckPatterns.fs#L598) where the compiler check a long identifier(described in the AST as `LongIdent`) case that has been resolved to a union case or F# exception constructor. So we know we are in corrent place.

```fsharp
SynMatchClause
    (LongIdent
        (SynLongIdent ([SayIt], [], [None]), None, None,
            Pats [Wild], None),
            None,
            Const
            (String ("Raise an issue on GitHub", Regular)))

```

We also know that we want to raise a compiler warning when we use an underscore “_” and that this is represented as `Pats[Wild]` in the AST. 


## Implementation

If you follow the logic of how patterns are checked you will see the following code:

```fsharp
// We can see that the compiler does not check for argument names on union types
let args, extraPatternsFromNames =
         match args with
         | SynArgPats.Pats args -> args, []
```

So we can add a check for SynPat.Wild and SynPat.Named based on if a union type has arguments names. But before we do that we need to find a clear an meaningful waring message ie : “Pattern discard is not allowed for union case that takes no data”

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
    // New compiler warning : “Pattern discard is not allowed for union case that takes no data”
    match msg with
    | SeeIt _ -> "Something that needs to be fixed"
    | SayIt _ -> "Raise an issue on GitHub"
    | SortIt _ -> "if you can, submit a pull request"

```

Now it is time to raise a [Pull request](https://github.com/dotnet/fsharp/pull/14055) to validate our fix. The compiler team will help and guide you through the process, ask you to cover more cases, testing to make sure the implementation is bullet proof.

In summary, We now have a new compiler warning that wil make our life easier when working with F#. But we can do better and implement a quick fix in our preferred IDE to make it even more easier for us.

## IDE support

For this we will use Jetbrains [Rider](https://www.jetbrains.com/rider/) as our editor of choice. This can be a whole new post about how I implemented the quick fix. At the time of this post [resharper-fsharp](https://github.com/JetBrains/resharper-fsharp) is not using the latest compiler version but our quick fix will also work with some small modifications.

![Quick fix](/images/union-case-quick-fix.gif)
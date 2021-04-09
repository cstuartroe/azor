# Azor

Azor is a programming language I made with the goal of being easy to implement but not completely trivial,
so that I can re-implement it as a way to learn new languages, try out new compiler design techniques, and
assess languages for use in implementing [other, more complex languages](https://www.github.com/cstuartroe/teko).

## Implementations

Currently, the only implementation is in [Python](https://github.com/cstuartroe/py-azor).

I have plans to attempt implementations in Rust as well as at least one of Haskell, OCaml, and/or Idris.

## Features

Azor is a statically-typed impure functional language. It is impure in the sense that it exposes three functions 
in the [standard library](#standard-library) which have side effects or whose output is not deterministic from the
set of arguments.

### Syntax

Azor declaration syntax is a little unusual. Declaring a constant (say, a tuple) looks like this:

```
two_things : (INT, BOOL) = (5, true)
```

Declaring a function looks like this, with arguments listed after the return type:

```
is_even : BOOL(n : INT) = n % 2 == 0
```

Expression syntax is a little less surprising. Azor has `let ... in ...` blocks, `if ... then ... else ...`, and uses `~` to join linked lists
in a similar fashion to Haskell `:`. All syntactic blocks in Azor are expressions, not control flow; Azor has nothing resembling imperative
programming.

Azor also uses `if` statements to branch on whether linked lists are empty:

```
all : BOOL(l : [BOOL])
  = if head ~ tail <- l
    then if head then all(tail) else false
    else true
```

Azor is not whitespace-sensitive. 

Check out [stdlib.azor](stdlib.azor) to see examples of Azor syntax.

### Type system

Azor only has two primitive types, `INT` and `BOOL`. It also has tuple, linked list, and function types.

For command line input and output, arrays of `INT`s are treated like strings, with integers corresponding to the character at the corresponding
Unicode code point. For example, `print([72, 105, 33])` will output `Hi!` to stdout.

Azor also supports string literals, which are simply evaluated to lists of integers. For example, the Azor literal `"Hi!"` evaluates to the
same thing as the expression `[72, 105, 33]`.

### Type inference

Azor does *not* have full Haskell-style type inference, but it does do some basic type deduction
on constants and function return types:

```
important_number = 42

square(n : INT) = n ** 2
```

Azor does not allow recursion or mutual recursion among implicitly typed functions, and implementations
should catch such violations and gracefully fail rather than entering an infinite loop. It should be
fairly apparent recursive type inference is not tractable in general:

```
foo = foo
```

This is an obviously unsolvable example, and there are more complex cases where a human or a better type checker would
be able to figure out the intended type:

```
triangular(n : INT) = if n == 0 then 0 else n + triangular(n - 1)
```

but I'm not going to try to optimize type inference any further.

### File structure

Azor files are simply a sequence of declarations. The order of declarations does not impact execution of the file in any way. 
Azor does not support modules, namespaces, or file imports.

For a file to be executable, it must declare a function called `main` of type `INT(args : [[INT]])`.
Executing an Azor file involves passing the command-line arguments to `main`, evaluating the function,
and exiting with its return value as the exit status.

Although the full file is type-checked prior to execution, evaluation is lazy, meaning that e.g. executing the following file would not
result in an infinite loop:

```
foo : BOOL = foo

main : INT = 0
```

### Generics

Azor does support generic functions. I was on the fence about this decision; in general, I tried to avoid including any advanced features
omitted by at least one major programming language, and I know that Go notably lacks generics. However, I've found this to be a frustrating
feature of Go and I think it would be even less tenable in a high-level functional language, so I decided to include generics after all.

Azor uses curly braces for generics:

```
map{A, B} : [B](f : B(A), l : [A])
  = if head ~ tail <- l
    then f(head) ~ map{A, B}(f, tail)
    else [] of B
```

I did make the implementation of generics a bit easier by requiring that generic functions be explictly supplied the type(s) when called:

```
is_evens : [BOOL] = map{INT, BOOL}(is_even, [1, 3, 5, 6])
```

### Features I have considered and (for the time being) declined to add

Deciding which features to include in Azor was a balancing act between making the language useful and making it easy to implement. 
I think the latter consideration should be more heavily prioritized for this project, so I tried to show restraint in adding features,
but I don't enjoy making a completely useless language so some features made it off this list (e.g., generics) or might at some point.

* Anonymous functions

* Currying

* Closures
  * I don't even think this would be possible unless I implemented namespaces/modules and/or imports first.

* Pattern matching

* Custom type definitions
  * I actually may do this as well - it wouldn't be too hard to implement and would make it a little easier to work with more complex
    data structures. I think the syntax would look something like this:

```
TYPE two_things_type = (INT, BOOL)
```

* Function caching
  * Since Azor is not a pure functional language, it could never be implemented with as much compiler trickery as a language like Haskell. In
    particular, because some functions like `print` have side effects, not all function return values can simply be cached. It should be 
    possible for an Azor compiler to statically analyze which functions make downstream use of impure functions, and cache any that don't. That
    said, I doubt I'll ever do it. It seems a little bit difficult, and too ripe for unexpected behavior.

* Imports and exports
  * Eh... maybe I will do this too, at least in a fairly basic way like C's `#include`. It'll make it substantially easier to organize projects
    in Azor.

* File I/O and other OS operations
  * I've actually declined to do this not because it would be hard to implement but because I don't want a hacked-together, potentially
    buggy language to be capable of side effects with real consequences. Maybe if I gain more confidence in the language over time I could
    add it, but honestly adding features like file I/O and networking seem like they push the language from something to write Oregon Trail
    in to something that I actually feel tempted to write real projects in, and I just don't think Azor is the language for that.


## Standard library

Most of the Azor standard library can be found in [stdlib.azor](stdlib.azor), which includes typical functional tools like `map`, `filter`,
and `reduce`; some list helpers like `zip`, `concat`, and `reverse`; some functions handy for printing like `println` (`print`, but it adds a newline),
`i2s` (integer to string), and `b2s` (boolean to string); and other odds and bobs like `all`, `any`, and `range`.

There are three standard library functions which cannot be implemented in Azor - these are the **impure functions**:

* `print` (type `()([INT])`): accepts a list of integers and [prints it](#type-system), returning an empty tuple `()`.
* `input` (type `[INT]()`): receives a single line of input from stdin.
* `rand` (type `INT(INT)`): accepts a single integer `n` and produces a pseudorandom integer between 0 and n-1. I leave the details of
  pseudorandom generation up to a library in the language of implementation.

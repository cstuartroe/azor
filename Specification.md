# A Specification of the Azor language

## File structure

An Azor file consists of a sequence of declarations. The order of declarations does not impact execution of the file in any way.

Azor does not support modules, namespaces, or file imports.

## Some general notes on syntax

Azor is not at all whitespace-sensitive.

All comma-separated sequences, including as function arguments, generic declarations,
lists, and tuples, permit a trailing comma with no change in meaning.

## Declarations

An Azor declaration can declare a constant of any type or a function.
Declared constants and declarations are available for use in the body of
all other declarations in the file regardless of the order they appear in
the file.

An Azor declaration has five components, some of which are optional: *name*,
*declaration of generics*, *type annotation*, *arguments*, and *body*.

### Name

This is just the name of global constant or function being declared, and
always comes first in a declaration.

Global names, like function argument names, local variable names,
and generic type names, must match the following regex: `[a-zA-Z_][a-zA-Z0-9_]*`.

Global names cannot be redeclared. That is, the following would result in an
error:

```
foo = 0
foo = 1
```

Names from the standard library also cannot be redeclared, e.g. the names `print`
and `map` are not available names.

### Declaration of generics

A function may declare one or more generic type labels to be used in the body,
which must be explicitly resolved to a type before the function can be used.

Generic type labels appear comma-separated in between curly braces, and must
match the same regex as other names. For example, the following are valid
generic declarations: `{A}`, `{n1, n2}`, `{a_type}`.

Generic declarations come immediately after the name. Generics can only
be declared on functions; attempting to declare generics on anything else
should result in an error. Azor implementations should not verify that
any or all declared generics are actually used in the body of a function.

### Type annotation and arguments

The type annotation is a syntactic element that describes the type of a
non-function, or the return type of a function, always preceded by a colon `:`.
For all constants and many functions, the type annotation is optional; however,
I frequently write it where not necessary for clarity.

All functions must also provide a comma-delimited, parenthesis-wrapped list
of named arguments that comes after the type annotation if both are present.
Each argument must be followed by a colon and a type annotation as well. The syntax
of all type annotations is described in the section on [type expressions](#type-expressions).

Here are some simple examples of type annotations:

```
n : INT = 40

f : INT(n : INT) = n + 5
```

Azor permits functions as arguments to other functions, so function arguments
may be annotated with a function type. However, Azor does not support anonymous
functions or any other strategy of creating functions in the body of a declaration,
so function types are not valid return types. For example, `INT(INT)` is a valid
argument type:

```
intApply : INT(f : INT(INT), n : INT) = f(n)
```

but the following declaration does not have valid type annotation, and cannot
be created in Azor:

```
nAdder : INT(INT)(n : INT) = ...
```

Azor attempts to perform type inference on all declarations. Type annotations
can be omitted for all constants, as their type can readily be inferred.
To understand where they can be omitted for functions, it's helpful to
understand the way Azor typechecks functions. In order to infer the return
type of a function, Azor typechecks its body. If the body of the function
contains a reference to a global variable, it will either use the type annotation
if present to determine that variable's type, or if the type annotation is not
present it typechecks the body of its declaration. If that declaration body contains
a reference to the function it was initially attempting to typecheck, then the
checker enters an infinite loop (from which it should be able to recover and
return an error).

This means that recursive functions must have return type annotation, and in
any chain of cyclically referential functions, at least one must have a type
annotation.

That is, this recursive function must have type annotations:

```
triangular : INT(n : INT) = if n == 0 then 0 else n + triangular(n - 1)
```

As well as at least one of this pair of mutually recursive functions:

```
isEvenSlow : INT(n : INT) = if n == 0 then true else !isOddSlow(n - 1)

isOddSlow(n : INT) = if n == 0 then false else !isEvenSlow(n - 1)
```

### Body

The body of a declaration is simply an [expression](#expressions) which evaluates
to the value of a constant or the return value of a function. All global variables
are in scope in the body of a declaration. Declared generics and function
arguments are also in local scope in the body of a function.

## Type expressions

### Simple types

There are two globally-scoped primitive types in Azor, `BOOL` and `INT`. Declared
generics also have the same syntactic behavior. All other types are built
from these atomic types

### List types

List types are denoted by square braces surround the element type of the list.
For example, `[INT]` means a list of integers, `[[INT]]` a list of lists of
integers, and `[([INT], BOOL)]` a list of tuples of type `([INT], BOOL)`.

### Tuple types

Tuples are fixed-width sequences whose members may be of different types. Tuple
type expressions consist of a comma-separated list of element types, surrounded
by parentheses. For instance, `(BOOL, INT)` is a valid tuple type. One expression
of type `(BOOL, INT)` is `(true, 1)`.

One-element tuple types are permitted in Azor - they must have a trailing comma:
`(INT,)`. The zero-element tuple type is also permitted: `()`.

### Function types

Azor type expressions may be followed by a single tuple of argument types,
indicating a function type with the given arguments and return type. The tuple
of arguments has the exact same syntax as tuple types. For instance,
`INT(INT)` describes functions which accept a single integer argument and return
an integer. The return type may be any non-function type, while argument types
may be anything, including function types.

For instance, the following are valid function types: `BOOL([INT])`, `[INT](INT, INT)`,
`[(INT, INT)]([INT], INT(INT))`, `()(INT(INT(INT)))`.

While function types cannot be return types, tuple types which contain function
types are permitted: `(INT(INT))(INT)`.

Functions may have an empty argument tuple, e.g. `INT()`, `()()`. The differences
between a constant and a function with no arguments is that the body of any function,
including one with no arguments, is reevaluated every time, which may be useful
if the function has side effects.

## Expressions

### Simple expressions

There are some single tokens which constitute an expression in Azor. `true` and
`false` have type `BOOL`. Integer literals match the regex `(0|[1-9][0-9]*)`
and have type `INT`. Identifiers have the type of whatever in-scope variable
they refer to, if any.

Character literals consist of a single Azor character surrounded
by single quotes, and have type `INT`. String literals consist of any number
of Azor characters surrounded by double quotes. Azor characters in character literals
and string literals may be a single alphanumeric or punctuation character other
than `\`, `'`, `"`, a space, or an escape sequence `\t`, `\r`, `\n`, `\\`, `\'`,
`\"`, with predictable value.

### Linked lists

The easiest way to express a linked list in Azor is a comma-separated sequence
of elements between square braces. All elements of a linked list must have the
same type. If and only if a linked list is empty, it must be followed by `of`
and a type annotation, e.g. `[] of INT`.

Values can be extracted from a linked list using an [if statement](#if-statements).

### Tuples

Tuples are syntactically similar to lists, except that they are wrapped in
parentheses rather than square brackets. Tuple elements may have different types.

Values can be extracted from a tuple using a [let statement](#let-statements).

### ~

Another way to construct linked lists is with the `~` operator, which links
a head to a tail, e.g. `0 ~ [1, 2, 3]` has the same value as `[0, 1, 2, 3]`.

### !

Expressions with type `BOOL` can be negated with a `!` prefix: `!true`.

### -

Expressions with type `INT` can be negated with a `-` prefix: `-10`.

### Binary operations

The comparators `==`, `!=`, `<`, `<=`, `>`, `>=` compare two expressions of
type `INT` and yield a `BOOL` value: `n == 0`.

The logical operators `&`, `|`, `^` (xor), and `!^` (boolean equal) operate on
two expressions of type `BOOL` and yield a `BOOL` value: `b & true`.

The arithmetic operators `+`, `-`, `*`, `/` (floor division), `%`, and `**`
(exponentiation) operate on two expressions of type `INT` and yield an `INT` value.

`~` and comparators have precedence 1, logical operators and `+`, `-`, `%` have
precedence 2, `*` and `/` have precedence 3, and `**` has precedence 4.

### Function calls

Expressions of function type can be called with a sequence of positional arguments,
which must match their declared argument types: `foo(true, 0)`.

### If expressions

`if ... then ... else ...` syntax can be used to branch on a boolean value:
`if n < 0 then -n else n`.

The same expression type can be used to branch on whether a linked list is
empty, attempting to split it into a head and tail, which also uses
the token `~`:

```
all : BOOL(l : [BOOL])
  = if head ~ tail <- l
    then if head then all(tail) else false
    else true
```

The new variable names created by unpacking a list (in this case, `head` and
`tail`) are not in scope in the `else` block of the expression.

### Let expressions

`let ... in ...` allows a value to be computed and named for use in another
expression:

```
foo(n : INT) = let nsq <- n ** 2 in (-nsq, nsq)
```

It can also be used to ensure an order of actions, which can be useful if
on or both actions have side effects, effectively imitating imperative
programming. In such cases I usually use variable names like `_` to clarify
that the stored value is not used:

```
println : ()(s : [INT])
  = let _ <- print(s) in
    print("\r\n")
```

`let` expressions can also be used to access individual tuple elements from a
constant or an expression with tuple type:

```
foo = (5, "Five")

bar = let (number, name) <- foo in
      println(name)
```

### Generic resolution

Function that were declared with generic type parameters must be explicitly
resolved to specific type or types before being called. This is most often
done right before being called:

```
map{A, B} : [B](f : B(A), l : [A])
  = if head ~ tail <- l
    then f(head) ~ map{A, B}(f, tail)
    else [] of B

isEven : BOOL(n : INT) = n % 2 == 0

isEvens = map{INT, BOOL}(isEven, [1, 2, 3, 4])
```

However, generic resolution can be performed anywhere:

```
mapIntsToBools : [BOOL]([INT]) = map{INT, BOOL}
```

## File execution

An Azor file must declare a function called `main` of type `INT(args : [[INT]])`.
Executing an Azor file involves passing the command-line arguments to `main`, evaluating the function, and exiting with its return value as the exit status.

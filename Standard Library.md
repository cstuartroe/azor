# The Azor standard library

This document attempts to explain and give examples of all standard
library functions. There are some helpers defined within the standard library,
which are not intended to be used on their own and are not described here but
may nonetheless cause name collisions.

## Notes on convention

For functions which may return no result (i.e., for situations that would
use the `Maybe` monad in Haskell), I've settled on the convention of
returning a list which has no elements if the operation was unsuccessful/returns
no result and has a single element if the operation does have a result.

## Impure functions

### print

Type: `()([INT])`

Accepts a string (list of integers) and prints it to stdout.

### println

Type: `()([INT])`

The same as `print`, but prints a newline after.

### input

Type: `[INT]()`

Accepts a single line of input from stdin and returns it as a list of integers,
without the trailing newline.

### rand

Type: `INT(INT)`

Accepts an integer n and returns a pseudorandom integer between 0 and n-1, inclusive.

## List utilities

### len{A}

Type: `INT([A])`

Returns the number of elements in a list.

### map{A, B}

Type: `[B](B(A), [A])`

Accepts a function and a list and applies the function to each element of the
list.

### filter{A}

Type: `[A](BOOL(A), [A])`

Accepts a boolean function and a list and filters out all elements which do
not satisfy the function.

### reduce{A, B}

Type: `B(B(A, B), [A], B)`

Accepts an updating function, a list, and a starting value, updates the
starting value using each element, and returns the final resultant value.

```
add(m : INT, n : INT) = m + n

sum : INT(l : [INT]) = reduce{INT, INT}(add, l, 0)
```

### zip{A, B}

Type: `[(A, B)]([A], [B])`

Accepts two lists and pairs corresponding elements into tuples. If one list
is longer than the other, the returned list has the same length as the
shorter input list.

### reverse{A}

Type: `[A]([A])`

Reverses the order of a list.

### concat{A}

Type: `[A]([A], [A])`

Concatenates two lists.

### list_eq{A}

Type: `BOOL([A], [A], BOOL(A, A))`

Given two lists and an equality function to compare individual elements,
evaluates whether the lists all elements of the lists are equal. Returns false
if the lists have unequal length.

### repeat{A}

Type: `[A](A, INT)`

Given an element and a non-negative integer n, creates a list of the element
repeated n times

### repeatF{A}

Type: `[A](A(), INT)`

Like `repeat`, but rather than accepting an element, accepts a function which
takes no arguments and runs it n times. The only reason to prefer this over
`repeat` is if the function is impure.

### at{A}

Type: `[A]([A], INT)`

Given a list and an index n, returns a list containing the nth element, or an
empty list if n is greater than the length of the list.

### index{A}

Type: `INT([A], A, BOOL(A, A))`

Given a list, an item, and an equality function, returns the position of
the first element of the list which is equal to the given item, or -1 if
no such element exists.

## String utilities

### i2s

Type: `[INT](INT)`

Returns a string (list of integers) representing a given integer, e.g.
`i2s(-43)` is `"-43"`.

### b2s


Type: `[INT](BOOL)`

Accepts a boolean, and returns `"true"` if it is true and `"false"` if it is false.

### l2s{A}

Type: `[INT]([A], [INT](A))`

Given a list and a function to convert each element of the list into a string,
converts the whole list to a string by including commas separating each element
and putting square braces on either end. For instance, `l2s{INT}([1,2,3], i2s)`
returns `"[1, 2, 3]"`.

### scat

Type: `[INT]([INT], [INT])`

A special case of `concat` for concatenating strings, e.g. `scat("Hello, ", "World!")`
returns `"Hello, World!"`.

The source code for `scat` is simply:

```
scat = concat{INT}
```

### sjoin

Type: `[INT]([[INT]], [INT])`

Given a list of strings and a string, returns a string consisting of the list of
strings interspersed with the given string, e.g. `sjoin(["a", "b", "c"], "-")`
returns `"a-b-c"`.

### parseInt

Type: `[INT]([INT])`

Accepts a string that should represent an integer, and if it is a valid representation
of an integer returns a list containing that integer, otherwise returns an empty
list. For example, `parseInt("-43")` returns `[-43]` and `parseInt("foo")`
returns `[] of INT`.

### rpad

Type: `[INT]([INT], INT, INT)`

Given a string, a desired length, and a character (integer) to append on the
right edge, if the string is less than the desired length this pads the given
character on the right until the string is the desired length, otherwise returns
the original string. For example, `rpad("a", 3, ' ')` returns `"a  "` and
`rpad("aaaa", 3, ' ')` returns `"aaaa"`.

## Other

### range

Type: `[INT](INT, INT)`

Given integer m and n, returns a list of the integers from m (inclusive) to n
(exclusive). E.g., `range(0, 5)` returns `[0, 1, 2, 3, 4]`.

### all

Type: `BOOL([BOOL])`

Given a list of booleans, returns true if they are all true. Returns true
for an empty list.

### any

Type: `BOOL([BOOL])`

Given a list of booleans, returns true if any of them are true. Returns false
for an empty list.

### find{K, V}

Type: `[V]([(K, V)], K, BOOL(K, K))`

Treats a list of 2-tuples as a key-value store with linear time search. Given a
list of 2-tuples, a key, and an equality function, for the first 2-tuple whose
first element is equal to the given key, returns a list containing
the second element of the 2-tuple.
Returns an empty list if no matching keys are found.

For example, if `d = [(1, "one"), (3, "three")]` and `intEq(m : INT, n : INT) = m == n`,
`find{INT, [INT]}(d, 3, intEq)` returns `["three"]` and `find{INT, [INT]}(d, 4, intEq)`
returns `[] of [INT]`.

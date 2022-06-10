---
title: "A New Programming Language (Part 1)"
date: 2022-05-14T18:42:03+02:00
draft: true
description: "This is the first of a series of posts on implementing a new programming language in Go."
---

This is the first of a series of posts on implementing a new programming language in Go.
<!--more-->

Throughout these posts, I will assume that the reader has some basic knowledge on the subject matter. For those new to this topic, I invite you to check out [Crafting Interpreters] by Robert Nystrom (there's a also free online version of the book). You will learn everything you need to know in order to understand what we'll be doing here.


## Some context

This is really a case of yak shaving[^yak-shaving]. Let's just say that I've always had a thing for programmable scientific calculators, and I haven't found any mobile phone application that does just that with a decent user experience. At least not on Android, and in terms of UX, emulators for actual devices, most of which being only of archaeological value[^free42], just don't cut it.

The closest I could find is the HP Prime emulator[^HPPrime]. However, despite the actual touch screen of the real thing, it's only an emulator and the UX is pretty poor compared to what should be achievable on a mobile phone.

So here we are, I want to build a programmable scientific calculator app and I need a programming language for it. Also, this will be a purely numerical platform, I do not need a computer algebra system or even symbolic math capabilities. Since I have not found an existing language that suits my needs and I've been itching to implement a programming language for a while, time to make one!


## The Cyon language

The tentative name for the app itself is Procyon, so I figured... Yeah, I'm the most creative person. And Yak was already taken.

Let's start with a brief aside on what I envision for the app UI and user experience.

The app will present the user with a classical layout: a (soft) keyboard in the lower portion of the screen and the display in the upper portion. The lower part of the keyboard will be static with the usual 4 to 5 rows x 5 columns with the essential keys for number entry, basic arithmetic and <kbd>Enter</kbd> keys, while the upper half (3-4 rows of 6 keys) will change dynamically based on context. The user should also be able to switch to a full alpha keyboard with gestures or something like that. The key idea here is that whatever the user is doing, anything they might need should be only one to two key presses away. In programming mode, this should include all syntactic elements, keywords, a shorthand for the local scope's variables as well (some auto-completion mechanism for longer identifiers and built-ins will go on the display part directly). The default UI should just be a REPL for the language itself.

Here are my requirements for the language:

- beginner friendly
- implied multiplication: `3(x+2)` should be seen as `3×(x+2)`
- a syntax that feels natural for maths:
    - simple variable declaration and assignment `x = 42`
    - a basic function definition should be as simple as `f(x) = 2x+3` with no weird syntax for the absolute neophyte
- dynamically typed with native support for decimal floats, complex numbers, matrices, (hash)maps, lists and strings.
- garbage collected
- compact syntax because of the small screen estate (30 columns of display at best, for readability and ease of interaction for editing purposes)
- [first class functions]

Here is a quick example of what I want the language to look like:

```
# this is a comment

# simple function definition
f(x) = 2x^2 + 3x - 7

# higher order function example
deriv(f@, x) =
    h = 10^-34
    = (f(x+h) - f(x)) / h

f(2)         # = 7
deriv(f, 2)  # ≈ 11
```

Note that we use indentation to define code blocks. I would have preferred actual delimiters for code blocks, but I went with identation for three reasons:

- small screen estate: indentation does not require a closing token (like `endif`, `end`, or `}`) thus reducing the line count.
- we're short on grouping symbols. As we'll see later, I had to use `{}` for indexing. Using it for code blocks would lead to ambiguous situations like `if cond { do_stuff }` forcing us to enclose `if` conditions in parentheses, which is not desirable.
- we want to keep the amount of keywords to a minimum, so no `do` - `end` constructs. The screen estate limitation also applies here.

From this point, I'll discuss some of the language features and their broader impact on the language design. I strived to maintain usability, sometimes at the cost of complexity in the compiler.


## Decimal floating point

Floating point numbers encoded in binary form, as used in most programming languages, are not well suited for a calculator for the sole reason that simple decimals like 0.1 cannot be encoded accurately with binary floats. Doing simple maths eventually drifts irremediably from the actual exact result. Sure one could clamp the displayed significant digits to 2 and get away with it. Only for a while.

Cyon uses IEEE-754 decimal floating point[^fp] where numbers are encoded as `sign × mantissa × 10^exponent` where sign is 1 or -1, mantissa ∈ [0,1) and exponent ∈ [-2147483648, 2147483647]. By default, the precision is 34 decimal digits but can be extended up to 2147483647 decimal digits (how to do so is yet to be determined).


## Implied multiplication and its direct implications

Implied multiplication can lead to ambiguous code, both for the user and for the parser. Here are a few of the challenges it poses and the adopted solutions, where I tried to make things as simple as possible for the user.


### Order of operations

Should `1/2√3` be seen as `(1/2)√3` or `1/(2√3)`? This has already been discussed at length elsewhere[^order]; the short story is that back in the days when typesetting did not allow what we call now "textbook" presentation of mathematical exprsssions, the rule was that implied multiplication had higher priority than explicit multiplication or division. While this made sense in that context, the general consensus nowadays is to apply the [PEMDAS] rule. So implied multiplication has the same priority than explicit multiplication and division in Cyon: `1/2√3` is `(1/2)√3`.


### Function application vs. multiplication

Distinguishing function application from implied multiplication is in fact the biggest issue. Consider this expression:

    1/f(x)^2

If `f` is a function, this is `1/((f(x))^2)`, and for any other type it is `(1/f)×(x^2)`. The AST is completely different depending on the type of `f`, and since Cyon is dynamicaly typed, this happens at runtime. Without puting much thought into it, I initially dismissed the idea of rebuilding the AST at runtime and went for a kind of static typing for functions. That was until I had almost everything else ironed out in the language, and it kept making things more complex for the user. Time to go back to the drawing board.

So I started thinking about efficient strategies for rebuilding the AST at runtime, not that hard with an interpreter, but with a VM I thought why not just jenerate bytecode that would do `if typeof(f) { do_this } else { do_that }`. Well, add a bunch of variables in a single expression and you end up with 2<sup>bunch</sup> branches... So not an option.

The AST really needs to be rebuild at runtime, but only for those ambiguous expressions. And how often does the type of a variable change in real code, especially in a way that it affects the AST? Never. Almost. So we don't need anything too fancy (like AST caches). For a future version, I think I'll build on this to dynamically generate optimized bytecode based on the actual value types in an expression.

TODO: talk about the actual implementaion: the basic idea is for ambiguous expressions to keep the AST around, rebuild on eval and keep a cached version of it. If the type of any involved vars change on next eval, rebuild.


### Array indexing

Another issue with implied multiplication is that using the commonly used `[]` both as a delimiter for matrixes and array indexing is ambiguous. `a[1,2]` could be either `a×[1,2]` or cell at line 2, column 3 of matrix `a`. The priority for Cyon is to keep `[]` for vectors and matrices. We're left with curly braces for indexing `{}`. 

Also, rebuilding the AST dynamically cannot help here.

While on the subject of indexing, indices start at 1. This is because it is standard pratcice to index matrix lines and columns starting from 1. We cannot start to treat other containers differently, and I do not want to impose 0-based matrix indexing.


## Type coercion

When an operator has operands of different types, one of the operands is converted so that there is as little information loss as possible. That is integers are converted to floats and floats to complex numbers as needed. Also, the division operator will convert integers to floats before performing the division. For integer division, one should use the integer division operator instead (`\`). Any built-in function that might return a float, like transcendental functions and nth-root, will convert integer arguments to floats as well.

Note that a converting a very large integer to a float will not necessarily prevent loss of information: if the float precision is clamped at 34 digits, an integer larger than 10<sup>34</sup>-1 will be truncated.

The goal is not to make for self-optimizing code, but to make type coercion more intuitive for the user.


## Expressions & statements

I wanted Cyon to feel functional whenever dealing with pure maths functions (one-liners), but take a more procedural[^procedural] approach as soon as control flow is required. As a result Cyon has both statements and expressions (and a proper ternary operator, reusing `if` and `else` like in Python).


## Constants

Cyon has the following built-in consants:

- `i`, which represents the imaginary unit `i = √-1`. 
- `π` (U+3C0), represents the number pi with an accuracy that matches the current floating point precision
- `pi`, equivalent to `pi`
- `e`, Euler's number with an accuracy that matches the current floating point precision

Making `i` and `e` constants may seem to be an odd choice but it is consistent with that writing a complex constant like `2+3i` is as simple as that (thanks to implied multiplication) and `2e^(π/2i)`. Another option would have been to use a different names, like `𝑖` (U+1D456). On a device with a dedicated keyboard (like an app), this is not an issue, but this is not viable on a desktop. Like `π` and `pi`, I could have used `𝑖` along with some other consant `_i`, but what about `e`? I think this would just make things more complex than they need to be.

A number of physical constants will be added later. These will be prefixed with an underscore (like `_c` for the speed of light). Although I have no plans to add support for units, this could also be a way to implement it (`_` is only allowed at the beginning of variable names).


## Names

By implied multiplication, we mean that `3f` is `3×f`, but what of `f3`? I have Not been able to find any reference material on how this must be dealt with, so let's see what's common practice:

- Xcas, Xcas-based CAS (HP Prime), and the Numworks[^numworks] calculator see `f3` as the variable `f3`
- [Geogebra] and the TI-84+ see it as `f×3`

The problem with the later is that it prevents the use of numbers in variable names. Also considering that humans seem to always write numbers first in this kind of expression, Cyon will adopt the former interpretation and see `f3` as a variable name. Should a user make the mistake of expecting `f3` to be an implied multiplication, it's fairly easy to implement helpful error messages (don't just throw a syntax error at their face!).

Now there's the issue of expressions like `e^(iπ)`. If we allow naming rules like in Go, a variable name can be made of any unicode letter. That is `iπ` is a valid Go variable name. If we want to allow `iπ` to be seen as `i·π`, a possible solution would be to limit variable names to a single script (i.e. `hello世界` would be seen as two distinct names `hello` and `世界`). Well, that's the only solution that I can think of, and it would only solve the problem for the specific case of `iπ`, not for `x·i` or `σ·π`. So Cyon will allow complete freedom in names and force either explicit multiplication or spaces between variable names, as in `e^(i π)`.


## Truth value testing

There is no actual boolean datatype. A boolean expression like `a > b` will result in the integer value -1 if true, 0 otherwise. Cyon has the usual bitwise operators `|`, `&`, `!` as well as short-circuit operators. I'm still not sure whether these will be like `||`, or use an `or` keyword. Since the modulo operator will be `mod` instead of `%`, I might end up using keywords for this too.

Ultimately, when an expression is converted as true or false (like in an `if` statement), the following rules apply:

Data type | Boolean value
----------|----------------
number    | n ≠ 0
container | len(c) ≠ 0
function  | error

There are no `true` or `false` keywords, use -1 and 0 instead.


## Variables and scoping

I'm strongly in favor of mandatory identifier declaration and lexical scoping like in Go. However, in Cyon, we want simple function declarations using the `=` sign:

    f(x) = x^3 - 2x^2

At first glance, this leaves us with `=` as the sole means to declare and assign variables. So what's the problem with this you may ask? Consider this bit of code (just ignore the fact that its Python code and anything you may know about scoping in Python):

```python
n = 1
def foo(x):
    n = 2
    return x + n
```

Should `n=2` rebind the `n` variable outside of `foo` or create a local variable `n`? There are two schools of thought here, both illustrated by how Python and Lua do this. In python, this creates a local variable, and blocks are function-wide (with all kinds of weirdness and tar pits that stem from that), and the way to force `n=2` to rebind the outer one is to use a `nonlocal` statement before that; nothing wrong with that, except that it's a declarative statement that cannot be combined with the assignment. In Lua, `n=2` just rebinds the outer one and one could make it local to the current block by just writing `local n=2`, with the ability to make a variable local to a loop body.

Before going further on the topic, we'll need a `del` statement in order to delete map keys, list items, and variables. Deleting variables is not a thing in most languages, but in Cyon's specific use case, the user will want to delete (global) variables when in interactive input. Note that a built-in `delete` function would not do here since `delete(x)` would get the value OF `x`, not the identifier itself. Just to keep things as sane as possible, we'll restrict the use of `del` for variables to globals. And we need globals by the way.

Back to scoping, for what I want to achieve with Cyon, the best way to handle it is the way Lua does: we have proper lexical scoping and `f(x) = foo` is valid code. We need to introduce a way to declare local variables. But wait, we can use `:=` for this! No new keyword and we can even use it in loop headers like `for i := 1..10`, and we have Go style loops where the index can be declared in the header as local to the loop body.

The main issue is that a variable not defined in the current scope is a global and that lookup of global variables is done at runtime. My only real gripe with this is that an expression `g(x) = 2f(x)` will compile, even if `f` has not been defined yet, but if we want to be consistent with support for global variablke deletion, we've got to live with it. As for the performance impact of dynamic lookup, it can be mitigated by making it static behind the scenes.


## Higher order functions and lists

Functions are defined like this:

```
funcName(arg1, arg2) =
    "docstring"
    funcBody()
    = retValue      # return
```

In the above example, `=` can be replaced by `:=` to define a local function.

There's also lambda expressions (aka function literal):

```
(arg1, arg2) -> single()expression()
```

Where parentheses can be removed for single argument functions. Note that because of the use of indentation to denote code blocks, like in Python, lambda expressions are limited to a single expression on the right hand side of `->`. If a more complex function is needed, the workaround is to define a local function and that instead of a lambda.

Regarding the grouping of arguments, consider this:

    foo(x, (y, z) -> bar(y, z), t)

Here, we group the arguments and have `->` take priority over `,` in the parser. We could instead decide that grouping is not needed, in which case the whole lambda expression needs to be parenthesized if we want to distinguish it from `foo`'s other arguments:

    foo(x, (y, z -> bar(y, z)), t)

which is much less readable with long expressions.

Here's a higher order function example:

```
map(sequence, fn) =
    result := ()                    #1
    for _, item := sequence:        #2
        result += (fn(item) ,)      #3
    = result                        #4

map((1,2,3), x -> 3x)               #5
```

Some explanations:

1. declare result as a local variable of `map` and initialize it to an empty list.
2. for loops are either infinite with the form `for:`, with a boolean exit condition `for booleanExpr:` or an iteration over a sequence with an assignment expression `index, item = sequence`. Here, we use a local assignment form, making item local to the loop's body.
3. two things here:
    - the `+` operator used for list concatenation. We could use something else, but I'm not sure what.
    - `(foo,)` is a list with a single item. That's to differentiate it from a perenthesized expression `(foo)`.
4. returns `result`. This is concise, and we don't need an actual `return` keyword. 
5. this just calls `map` with a list and a lambda expression as a function.

For list delimiters, I could use something like curly braces instead and get rid of the comma for singletons, but is there any real benefit? On a calculator keyboard, `()` would be direct keys, while `,` and `{}` would likely be shifted. On the other hand, using `{}` would probably make list manipulation less prone to user mistakes. Another thing to consider is that for maps[^maps] I'd like to re-use the same delimiter as for lists and have a syntax like `(key: value)`. `:` is the potentially problematic bit here.


## Deferred execution and error handling

These two topics are tied together. Error handling is most often done by either returning error values or using exceptions. Note that languages that have exceptions can usually do both. Which is best is debatable although I prefer using error values despite the fact that it tends to make for more verbose error handling code. Well, for Cyon that's actually the issue, we want compact code, so we'll go for exceptions. The typical construct is (pseudo python-ish code):

```python
try:
    f = open(file)
    data = read(f)
except err:
    raise err
finally:
    if f != nil:
        close(f)
```

What's nagging at me here is the finally block where we still need to check if `f` is set before closing it. What if we had exceptions and deferred execution?

```python
try:
    f = open(file)
    defer close(f)
    data = read(f)
except err:
    raise err
```

Add in a bunch of other resources that must be acquired and released in sequence, and it makes the code a lot cleaner. Admitedly, it does not save much in terms of keystrokes, and we've traded a `finally` keyword for a `defer`, but `defer` has more uses. 


### Defer statement

A `defer` statement defers the execution of the statement or block following it to the moment the surrounding function returns, either because the surrounding function executed a return statement, reached the end of its function body, or because an exception occurred.

```ebnf
DeferStmt           = "defer" [ LocalAssignmentStmt ] ":" Continuation .
LocalAssignmentStmt = Identifier { "," Identifier } ":=" Expression { "," Expression } .

Continuation        =  StmtList newline | newline indent Statement { Statement } detent
Statement           =  StmtList newline | CompoundStmt
StmtList            =  SimpleStmt { ";" SimpleStmt } [ ";" ]
```

Deferred blocks are invoked immediately before the surrounding function returns, in the reverse order they were deferred. That is, if the surrounding function returns through an explicit return statement, deferred blocks are executed after any result parameters are set by that return statement but before the function returns to its caller.

An explicit return statement used in a deferred block will change the function's return value, stop execution of the current block and resume at the next deferred block.

Go's `defer` statements use a function for which the arguments are evaluated immediately, but the actual function invocation is deferred. Because we cannot write multiline function literals (lambdas) in Cyon, doing the same would lead to ugly things like:

```
foo() =
    bar(x) := 
        # bar's body
    defer bar(1)
}
```

So the strategy here is to make `defer` a compound statement with an optional local assignment:

```
f := open("some file")
defer t := f: close(t)      # equivalent to Go's "defer close(f)", we bind `f` to a local `t`.
```

Where the assignment is executed immediately. Note that:
- `defer: close(f)` would be equivalent to `defer func(){close(f)}()` in Go.
- I used different variable names `t` and `f` for clarity, but `defer f := f: close(f)` would have worked just the same: the `f` of the left hand side of the assignment shadows the one on the right hand side.
- I haven't mentionned this yet, but assignments accept multiple targets just like in Go: `a, b, c := d, e, f`.


### Error handling

Alright, we know that we want to use exceptions because it tends to make for more concise code. Not only we get rid of tons of `if err != nil {}`, when calling deeply nested functions, those in the chain that do not need to care about errors get exited automatically by the underlying exception handling mechanism.

Exceptions are to handling of exceptional conditions casued by factors external to the programe, like filesystem access or error in user input. Runtime errors, like an invalid index, are not catchable. The err() builtin function returns the exception stack as a list of error, code location pairs.

Cyon does not introduce a catch keyword, but uses `elif` and `else` instead:

```
foo(file) =
    # foo returns the contents of file or some
    # default data if the file doesn't exist
    data := ()
    try:
        f = open(file, "w+")
        defer f:=f: close(f)
        data = parse(f)
    elif e := err(); e.0 == _ErrNotExists:
        = (some, default, data)
    else:
        raise
    = data
```

TODO: once done, talk about the implementation of lexer/parser, give the nQueens example and link to the spec.


[^yak-shaving]: https://en.wiktionary.org/wiki/yak_shaving and https://yakshav.es/the-patron-saint-of-yakshaves/
[^HPPrime]: Dirt cheap compared to the actual device. Hats down to HP for releasing this app BTW.
[^free42]: Worth mentioning at this is [Plus42] which started as an emulator for the HP-42s and has evolved far beyond the capabilities of the original hardware. The RPN buffs should definitely check it out, the desktop version is free and open-source and makes for an excellent desktop calculator.
[^fp]: There are other alternatives to decimal floating point, like [Recursive Real Arithmetic](https://dl.acm.org/doi/abs/10.1145/3385412.3386037) as used in the default Android calculator, but there are issues that make it impractical in a programming language.
[^procedural]: While there are valid arguments for both functional and procedural languages as starter languages, I belive that purely functional languages are not well suited for a scientific calculator (immutability would be an issue).
[^order]: If you do a Google search for `8+2(2+2)`, you'll see that this has become a meme.
[^numworks]: Numworks is a new player on the graphing calculator market. There's a demo [here](https://www.numworks.com/simulator/) and a free demo app on the Google Play Store, which unfortunately does not save state. 
[^maps]: I cannot use Go's built-in map as-is since most Cyon values cannot be used as map keys. I need to implement proper hash functions for all types and have the value comparison running before I get to that. This is the first thing on my TODO list as soon as I have a running interpreter.
[^todp]: Also known as a [Pratt parser].

[Crafting Interpreters]: http://craftinginterpreters.com/
[Plus42]: https://thomasokken.com/plus42/
[nQueens]: https://www.cl.cam.ac.uk/~mr10/backtrk.pdf
[first class functions]: https://en.wikipedia.org/wiki/First-class_function
[PEMDAS]: https://en.wikipedia.org/wiki/Order_of_operations#Mnemonics
[Geogebra]: https://www.geogebra.org/
[cyon-spec]: https://github.com/db47h/cyon/blob/main/docs/spec.md
[Pratt parser]: https://en.wikipedia.org/wiki/Operator-precedence_parser#Pratt_parsing

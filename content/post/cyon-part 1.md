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
- implicit multiplication: `3(x+2)` should be seen as `3×(x+2)`
- a syntax that feels natural for maths: a basic function definition should be as simple as `f(x) = 2x+3` with no weird syntax for the absolute neophyte.
- dynamically typed with native support for decimal floats, complex numbers, matrices, (hash)maps and strings.
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
- we want to keep the amount of keywords to a minimum, so no `do` - `end` constructs.

What follows a draft of the language specification and it will very likely change with experimentation and user feedback. Also, I do not mind adding complexity to the compiler or interpreter as long as it makes for a better user experience.


## Implicit multiplication and its implications

Implicit multiplication can lead to ambiguous code, both for the user and for the parser. Here are a few of the challenges it poses and the adopted solutions, where I tried to make things as simple as possible for the user.

### Function application vs. multiplication

My initial plan was to have only anonymous functions and make `f(x) = foo` syntactic sugar for `f = x -> foo` where `x -> foo` is the anonymous function. The problem with that is that the type of `f` would be only be known at runtime, thus making implicit multiplication ambiguous in expressions such as `f(x+1)`. This could be either `f × (x+1)` or a call to function `f` with `x+1` as its argument (function application). 

 Checking at runtime if the value of `f` is a function or some numeric type is a bad idea because of operator precedence. If we consider the expression `f(x+1)^3`, and `f` is a function, this must be executed as `(f(x+1))^3`, but if `f` is a number, this must be executed as `f*((x+1)^3)`, thus changing the AST dynamically. Likewise, `a(2)a(4)` could be `((a*2)*a)*4` or `(a(2))*(a(4))`.

 The solution I went for is to turn `f(x) = foo` into a static function declaration. `f` becomes statically typed, so from the compiler's point of view the expression `f(x+1)` is a function call. This also allows the compiler to bind `f` statically.
 
 This does not solve the issue for higher order functions however. Suppose a higher order function like:

    inverse(f, x) = 1/f(x)

The function parameter `f` is dynamically typed, so `f(i)` is ambiguous again. One solution to this is to introduce an explicit function application operator, but since we've already introduced static typing for functions, I belive that the most elegant way to do this is to add a type-hint mechanism. We do this by adding a `@` suffix[^suffix] to the identifier, either in its declaration (i.e. the function parameter name or first assignment), or at the call site:

    inverse(f@, x) = 1/f(x)
    inverse(f, x) = 1/f@(x)

Allowing the suffix at the call site simplifies the use of functions as values in tuples or maps:

    add2(x) = x+2
    add4(x) = x+4
    fns = (add2: add2, add4: add4)
    fns.add2@(1)

Without this, we would have to use a temp variable `t@ = fns.add2`, then call it, or write a wrapper function. While Cyon is definitely not designed with OOP in mind, this is still a nice option to have.


### Array indexing

Another issue with implicit multiplication is that using the commonly used`[]` both as a delimiter for matrixes and array indexing is ambiguous. `a[1,2]` could be either `a×[1,2]` or cell at line 2, column 3 of matrix `a`. The priority for Cyon is to keep `[]` for vectors and matrices, and since parentheses are not an option here, we're left with curly braces for indexing `{}`.


## Data types

Cyon is dynamically typed and garbage collected. While dynamic typing is not necessarily beginner friendly, the main reason for it is that it makes for less verbose code in most situations.

The following EBNF productions will be used throughout of this section as building blocks for literals:

```ebnf
letter        = unicode_letter .
decimal_digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" .
binary_digit  = "0" | "1" .
octal_digit   = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" .
hex_digit     = decimal_digit | "A" | "B" | "C" | "D" | "E" | "F" | "a" | "b" | "c" | "d" | "e" | "f" .
```

### Integers

An integer is any integer number, with up to 2^32-1 digits of precision.

For integer literals, the prefixes `0b`, `Oo` and `0x` may be used to set the base to binary, octal or hexadecimal respectively. These prefixes should not interfere with implicit multiplication. Underscore characters `_` may be used to separate digits for readbility.

```ebnf
int_lit        = decimal_lit | binary_lit | octal_lit | hex_lit .
decimal_lit    = "0" | ( "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ) [ [ "_" ] decimal_digits ] .
binary_lit     = "0" ( "b" | "B" ) [ "_" ] binary_digits .
octal_lit      = "0" [ "o" | "O" ] [ "_" ] octal_digits .
hex_lit        = "0" ( "x" | "X" ) [ "_" ] hex_digits .

decimal_digits = decimal_digit { [ "_" ] decimal_digit } .
binary_digits  = binary_digit { [ "_" ] binary_digit } .
octal_digits   = octal_digit { [ "_" ] octal_digit } .
hex_digits     = hex_digit { [ "_" ] hex_digit } .
```

    x = 1234567890122546879513216547896516
    print(x+1) // prints 1234567890122546879513216547896517


### Decimal floating point

Floating point numbers encoded in binary form, as used in most programming languages, are not well suited for a calculator for the sole reason that simple decimals like 0.1 cannot be encoded accurately with binary floats. Doing simple maths eventually [drifts irremediably](https://go.dev/play/p/-hl7zzAp8Fw) from the actual exact result. Sure one could clamp the displayed significant digits to 2 and get away with it. Only for a while.

Cyon uses IEEE-754 decimal floating point[^fp] where numbers are encoded as `sign × mantissa × 10^exponent` where `sign` is 1 or -1, `mantissa ∈ [0,1)` and `exponent ∈ [-2147483648, 2147483647]`. By default, the precision is 34 decimal digits but can be extended (how to do so is yet to be determined).

```ebnf
float_lit         = decimal_float_lit .

decimal_float_lit = decimal_digits "." [ decimal_digits ] [ decimal_exponent ] |
                    decimal_digits decimal_exponent |
                    "." decimal_digits [ decimal_exponent ] .
decimal_exponent  = ( "e" | "E" ) [ "+" | "-" ] decimal_digits .
```

    x = 1.2
    y = 1.17e27


### Complex numbers

The identifiers `𝒊` (U+1D48A) and `_i`[^i] are constants both representing the complex number `i`. A complex number is simply written as `real + imag × 𝒊` where the real and imaginary part are decimal floating point:

    x = 1 + 2_i

is the same as:

    x = 1 + 2𝒊

And `x_i` is `x × 𝒊`:

    print(x_i) # prints -2+𝒊
    print(x𝒊)  # prints -2+𝒊

TODO: we cannot have 𝒊 has part of an identifier. 


### Strings

Same as dobule-quoted Go strings. No support for backtick delimiters.

TODO: Develop


### Tuples

    ()                      # empty tuple
    (three, element, tuple)
    (x)                     # not a tuple
    (x,)                    # single element tuple
    t := (42,1); print t.0  # prints 42

Parentheses not needed except for the empty tuple.

TODO: concatenation op. Need `1 . (1, 2) => (1, 2, 3)` and `(1, 2) . (3, 4) => (1, 2, 3, 4)`. Which means `((1, 2), ) . (3, 4)` => `( 1, 2, 3, 4)`? Better semantics with `(1,) . (2, 3) => (1, 2, 3)`.

TODO: talk about operator rules for `scalar op (tuple)`, `(tuple) op (tuple)`, `op (tuple)`...


### Matrixes

A matrix literal is surrounded by square brackets, values in the same row are separated by a comma, and rows are separated by semi-colons:

    vec := [1; 2; 4; x] # column vector
    M := [ 1, 2, 3;
           3, 4, 7;
          -2, 1, 3 ]
    empty_Mat := []

Matrix cells can only contain numbers. For a generic purpose array-like construct, use tuples.


### Maps

    foo := (a: 42, b: 163)
    bar := ("more complex key": "string value", foo: "not foo above")

    print foo{"a"}
    print foo.a
    print bar{"more complex key"}
    bar{"newkey"} = value

TODO: develop

### Ranges

    1..5 // shortcut for (1, 2, 3, 4, 5) but internally uses less memory


### Functions

TODO: Develop this and limit lambdas to one liners => less messy syntax.

Functions are first class values. 

Formal function declaration:

    f(x) = 3x+7
    g(x) = f(x) - 7
    f(2)  # 13
    g(2)  # 6

Anonymous functions:

    a = x -> 2x
    a.(2)
    adder(v) =
        obj = {}
        obj = { "value": v, "add": n -> obj.v + n }
        return obj
    a = adder(2)
    a.add.(4) # 6

Functions not declared using the form `func_name(args) = func_body` must be called using the explicit function application syntax `identifier.(args)`. For more information, see the section about implicit multiplication and its implications.


## Type coercion

When an operator has operands of different types, one of the operands is converted so that there is as little information loss as possible. That is integers are converted to floats and floats to complex numbers as needed. Also, the division operator will convert integers to floats before performing the division. For integer division, one should use the integer division operator instead (`\`). Any built-in function that might return a float, like transcendental functions and nth-root, will convert integer arguments to floats as well.

The goal is not to make for self-optimizing code, but to make type coercion more intuitive for the user.


## Expressions

I wanted Cyon to feel functional whenever dealing with pure maths functions (one-liners), but take a more procedural[^procedural] approach as soon as control flow is required. As a result Cyon has both statements and expressions (and a proper ternary operator, reusing `if` and `else` like in Python).

Cyon supports the standard binary operators for addition, subtraction, ...

TODO

## Statements

TODO


## Constants

Cyon has two built-in constants:

- `𝒊` (U+1D48A): the complex number `i`. This is meant for use on devices with a dedicated keyboard.
- `_i`: the complex number i.

A number of physical constants will be added later, but these will be prefixed with an underscore (like `_c` for the speed of light) and `_` is not otherwise allowed in identifier names so that adding new constants will not break existing code. Although I have no plans to add support for units, this could also be a way to implement it.

User defined constants are not supported yet, but this is something I'm considering and it will probably use the same prefix mechanism instead of introducing a new keyword.


## True and false

There is no actual boolean datatype. A boolean expression like `a > b` will result in the integer value -1 if true, 0 otherwise. Complex boolean expressions can be build by combining basic boolean expressions with the bitwise operators `|` for `or`, `&` for and, `~` for not. For example:

    if (c >= 65) & (c < 65+26):
        # do stuff

Ultimately, when an expression is converted as true or false (like in an `if` statement), the following rules apply:

Data type | Boolean value
----------|----------------
number    | n ≠ 0
string, tuple, matrix, map | len(v) ≠ 0
function | error

There are no `true` or `false` keywords, use -1 and 0 instead.


## Variables and lexical scoping

TODO: talk about identifier names (what's allowed and not, like `_`)

While I'm strongly in favor of mandatory identifier declaration and lexical scoping like in Go, some desired features for the language prevent us to do so. For example, we want simple function declarations using the `=` sign:

    f(x) = x^3 - 2x^2

This prevents us from using Go-style declarations, using for example `:=`, as well as a pure lexical scoping. Using `=` for functions and `:=` for variables would not be consistent, so we're left with `=` as the sole means of variable declaration and assignment. Let's use another example to illustrate variable scope:

    fact(n) = 
        p = 1
        iter(n) = 
            if n <= 1:
                return p
            p = p*n
            return iter(n-1)
        return iter(n)

`fact` is just the classical recursive factorial that is modified to allow [tail call][tailcall] optimization. In this (non-working) example the programmer's intent is clearly to have `p` and `iter` defined only within the scope of `fact`. We also want `iter` to be a closure capturing `p`. So if we follow with the same logic, the assignment `p = p*n` should in fact declare a `p` that is local to `iter`.

If we tried the same in Python (which has similar scoping rules), this example would fail with an error:

> UnboundLocalError: local variable 'p' referenced before assignment

This is the correct behavior: `p*n` is evaluated first, using `p` declared in `fact` and then try to assign it to a `p` local to `iter`. To work around this, Python has the `nonlocal` statement. This is just too long and verbose for Cyon. The most compact and unambiguous way I've found to explicitely indicate the intent of modifing a non local variable is to add it to the function's parameter list, but separating it from other parameters with a semicolon:

    iter(n; p) = ...

This extends the formal function definition syntax to (in pseudo EBNF):

    func_def   = identifier '(' [args] [';' bound_vars] ')' '=' function_body
    args       = identifier { ',' identifier }
    bound_vars = identifier { ',' identifier }

**TL;DR**: Function and variables are bound at the function level. Their names are looked up in the current function's bound variables: function arguments and explicitly bound names from the enclosing scope, explicitly initialized variables and local functions. If that fails, the search continues recursively in the enclosing scopes (free variables).

When a variable is first assigned within a function:
1. If the variable is a free variable (i.e. from an implicit closure) and was referenced before assignment: error, suggesting an explicit closure
2. If the variable is not bound yet, bind it in the current function
3. Regular assignment


## Operator precedence

From highest precedence, to lowest:

Precedence     |  Operator
---------------|----------
function call  | (
index op       | . {
exponentiation | ^ !
product        | *  /  \\ %  <<  >>  &  
addition       | +  -  \|  ~
comparison     | ==  !=  <  <=  >  >=
concatenation  | , ;
anon function  | ->
assignment     | =

Note that the compiler uses a top-down operator precedence parser[^tdop], where the tokens '(' and '{' are infix operators. Also note that there are no boolean operators, only bitwise operators.


## Ternary op

We reuse the `if` and `else` keywords:

    x = value if cond else other_value

`else` is mandatory.


TODO: free variables in a function are not mutable. While this is probably trivial for most data types, we should be careful with library functions that mutate things like maps and tuples (i.e. make sure that no library function mutates a value). Also when doing `foo{x} = value` apply the same rules as for `foo = value`, i.e. if foo is not in the current scope, error. 

TODO: the above rule might be problematic if we want dynamically sizable matrices. Need a way to get that.


[^yak-shaving]: https://en.wiktionary.org/wiki/yak_shaving and https://yakshav.es/the-patron-saint-of-yakshaves/
[^HPPrime]: Dirt cheap compared to the actual device. Hats down to HP for releasing this app BTW.
[^free42]: Worth mentioning at this is [Plus42] which started as an emulator for the HP-42s and has evolved far beyond the capabilities of the original hardware. The RPN buffs should definitely check it out, the desktop version is free and open-source and makes for an excellent desktop calculator.
[^fp]: There are other alternatives to decimal floating point, like [Recursive Real Arithmetic](https://dl.acm.org/doi/abs/10.1145/3385412.3386037) as used in the default Android calculator, but there are issues that make it impractical in a programming language.
[^tdop]: Also known as a [Pratt Parser](https://en.wikipedia.org/wiki/Operator-precedence_parser#Pratt_parsing)
[^repl]: function identifiers must be mutable in the REPL: if the user writes `f(x) = 2x`, then `g(x) = .5f(x)`, they should rightfully expect to be able to modify the definition of `f` without having to rewrite `g`. While this could be allowed in a program, this is an obvious recipe for disaster.
[^suffix]: Some variants of BASIC (the good ones, not VB) were staticaly typed and used suffixes to indicate a variable's type.
[^procedural]: While there are valid arguments for both functional and procedural languages as starter languages, I belive that purely functional languages are not well suited for a scientific calculator (immutability would be an issue).
[^i]: We cannot simply use the form `1+2i` like in Go because of implicit multiplication. I thought about making `i` a constant, but that would prevent the user from using it as a variable. The app will have a dedicated `𝒊` key, so this is not really an issue, and on desktop, using `_i` instead is not too much of a burden in a text editor, and on the desktop version of the app, we could map `𝒊` to something like <kbd>Alt</kbd>+<kbd>i</kbd>.

[Crafting Interpreters]: http://craftinginterpreters.com/
[Plus42]: https://thomasokken.com/plus42/
[nQueens]: https://www.cl.cam.ac.uk/~mr10/backtrk.pdf
[tailcall]: https://en.wikipedia.org/wiki/Tail_call#Example_programs
[first class functions]: https://en.wikipedia.org/wiki/First-class_function
[dot notation]: https://en.wikipedia.org/wiki/Function_(mathematics)#Dot_notation
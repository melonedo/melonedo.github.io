---
layout: post
title: Julia Interpolation Cookbook
date: 2022-01-27 10:35:00 +0800
tags:
- 编程
- Julia
---

Interpolation is essential to metaprogramming in Julia, this post covers some of the tricky cases when interpolating exressions that I encounter.

# Official manual

It is expected that you read the [Program representation](https://docs.julialang.org/en/v1.7/manual/metaprogramming/#Program-representation) and the [Expressions and evaluation](https://docs.julialang.org/en/v1.7/manual/metaprogramming/#Expressions-and-evaluation) sections in the official manual about [metaprogramming](https://docs.julialang.org/en/v1.7/manual/metaprogramming/) before reading the content below. Also, the article about [Julia ASTs](https://docs.julialang.org/en/v1.7/devdocs/ast/) in the manual is a good reference when you compose `Expr` objects by hand.

# How to do [splatting interpolation](https://docs.julialang.org/en/v1.7/manual/metaprogramming/#Splatting-interpolation)?

In short, use `$(exprs...)`. But this isn't really that easy. You must make sure *splatting is interpolated in the right context*, since the interpolated expressions are the same in every context.

For example, suppose `exprs = (:a, :b, :c)`, `:($(exprs...))` will result in syntax error because Julia does not understand what you want. Below is some common use cases:

```julia
julia> exprs = (:a, :b, :c)
(:a, :b, :c)

julia> :($(exprs...);) # list of expressions
quote
    a
    b
    c
end

julia> :($(exprs...),) # tuple
:((a, b, c))

julia> :(; $(exprs...)) # named tuple. (;a,b,c) is short for (a=a,b=b,c=c)
:((; a, b, c))

julia> :[$(exprs...)] # vector
:([a, b, c])

julia> :[$(exprs...);] # also vector
:([a; b; c])

julia> :[$(exprs...);;] # row vector (matrix)
:([a;; b;; c])
```

Note currently keyword arguments (and named tuple) can not be constructed with interpolation unless you use the short form above, so to construct a general keyword argument, you must use the `Expr` way:

```julia
julia> f = :(foo(1,2))
:(foo(1, 2))

julia> insert!(f.args, 2, Expr(:parameters, Expr(:kw, :a, 3)));

julia> f
:(foo(1, 2; a = 3))
```


# How to include a literal `Symbol` in quotes?

```julia
julia> :($(Meta.quot(:a)) + b)
:(:a + b)
# or
julia> :($(QuoteNode(:a))+b)
:(:a + b)
```

They do result in different ASTs, but the difference is not significant for a single symbol.

# How to include a `$` in quotes?

The "uninterpolated" quote `$a` as in `:($a + b)` is valid input for macros, but is not documented. Let's examine what it is first:

```julia
julia> a=1
1

# does not work
julia> :($a)
1

julia> macro dump1(ex) ex end
@dump1 (macro with 1 method)

# because `$a` is not valid expression alone
julia> @dump1 $a
ERROR: syntax: "$" expression outside quote around REPL[94]:1
Stacktrace:
 [...]

julia> macro dump2(ex) dump(ex) end
@dump2 (macro with 1 method)

julia> @dump2 $a
Expr
  head: Symbol $
  args: Array{Any}((1,))
    1: Symbol a
```

So we see that `$a` is `Expr(:$, :a)`.

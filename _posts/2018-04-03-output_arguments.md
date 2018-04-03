---
layout: post
title: "Input-output arguments: reference, pointers or values?"
tags: [cpp]
description: >
  "How should a function handle in-out arguments?" is an old question that still come back from
  time to time. There's a lot of advice out there, yet the solutions are still debated.
  In this post I try my hand at offering an alternative.
author: mropert
customjs:
 - https://platform.twitter.com/widgets.js
---

A couple weeks ago, [Arvid Gerstmann](https://twitter.com/ArvidGerstmann) made a (somewhat innocent :D)
remark on twitter that sparked some debate:

<blockquote class="twitter-tweet" data-lang="en">
<p lang="en" dir="ltr">Never use non-const references as function arguments. Ever.</p>
&mdash; Arvid Gerstmann (@ArvidGerstmann)
<a href="https://twitter.com/ArvidGerstmann/status/977186597266477056?ref_src=twsrc%5Etfw">23 mars 2018</a>
</blockquote>

The follow-up debate generated quite a few answers and retorts on the matter, showing that while
not everybody agreed with Arvid's position, the general advice of "use non-const reference for in-out
arguments" didn't convince everyone either.

That piece of wisdom didn't come from nowhere, in fact it's written in the
[C++ Core Guidelines](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#f17-for-in-out-parameters-pass-by-reference-to-non-const):

> ### F.17: For “in-out” parameters, pass by reference to non-const
>
> **Reason** This makes it clear to callers that the object is assumed to be modified.

But if it's still widely debated, maybe it's because no satisfactory solution has been found so far.

I'll try to recap the various arguments for each case before offering my own suggestion
that goes in a different direction.

### Pointers

Pointers are the historical choice, coming from C where references weren't even an option.

```cpp
void foo(Bar* b);

Bar b;
foo(&b);
```

It has the advantage of making the pass-by-pointer explicit. When reading or reviewing the code, it's
very clear that `foo()` may very well make changes to the given `Bar` object.

But being a pointer, it also has the problem of introducing an extra concern: whether or not the argument
can be `NULL` (or `nullptr` C++). That's something the maintainer of `foo()` will need to clarify
and document, but can't enforce at compile time. Which may yield to a failed assertion or null-dereference
in production.

### References

The recommended C++ way (according to the core guidelines) is to use non-const references:

```cpp
void foo(Bar& b);

Bar b;
foo(b);
```

The advantage is that we do away with all the pointer semantics, inside `foo()` the given object
will behave like a value. There won't be a need to make a null check, and the compiler can enforce
that `b` is a valid object at compile time in most cases (the exception being if you pass a dereferenced
pointer that may in turn be null).

The drawback is the lack of explicitness in the code. A reader looking only at the caller side may miss the fact
that `foo()` takes the argument by reference and not expect it's own copy to be modified.
As an ex C programmer, I can confirm first-hand that this is terribly unsettling at first. For quite
some time I used pointers because I thought the lack of explicitness was terrible (and I believe
this is the rationale behind Arvid's initial tweet).

### `gsl::not_null`

The [Guidelines Standard Library](https://github.com/Microsoft/GSL) introduced a new construct named
`not_null<T>` which is essentially a pointer guaranteed to be non-null.

```cpp
void foo(gsl::not_null<Bar*> b);

Bar b;
foo(&b);
```

It has the advantage of the pointer explicitness with `&` operator and ensures that the argument
won't be null with a mix of assertions and compile-time checks. As the argument type is called `not_null`,
there's no debate or documentation needed to know that `foo()` expects a non-null pointer.

It still has the drawback that `foo()` implementation will have to manipulate the argument with pointer
semantics, adding extra boilerplate of `*` and `->` where the developer probably only wanted a value
that could be modified.

### `output_parameter`

On his [blog](https://foonathan.net/blog/2016/10/26/output-parameter.html), Jonathan Müller proposed
a new construct named `output_parameter` which is essentially a wrapper around a reference (not
to be confused `std::reference_wrapper`) with an explicit constructor:

```cpp
void foo(output_parameter<Bar> b);

Bar b;
foo(out(b));
```

Here the explicit call to `out` (a helper function to create an `output_reference<T>` from a `T&`)
clearly expresses intent, and the `output_parameter` argument can be assigned from another value,
updating the original object.

Alas, since it's not possible overload operator `.` in C++, the maintainer of `foo()` will still
need some boilerplate to call methods on the object, like using a `.get()` call first.

## Let's take a step back

When looking at alternatives to find the solution to a problem, it's easy to get drawn in so far
that you loose track of the issue you're trying to solve in the first place.

So first, let's ask the question: what issue are we trying to solve with input-output parameters in the
first place? Indeed, some languages do not even have that notion, and yet they are perfectly
able to work with a wide range of problems. They usually do that by taking all inputs by value
and return all outputs by value.

Believe it or not, when I wrote this article I had to double check if you can pass and return
structs by value in C. Turns out you can (even in C89), yet I can't recall a single time when I did
that. But I do recall being told since the beginning that you couldn't or shouldn't do that. Since you
can't overload `operator=` in C, that would make sense for some types, but not even all.

In C++, you can obviously pass any type by value (as long as it's copy-constructible) but it's
still discouraged, since you incur a copy cost. It's also more painful to return multiple values
because you need to wrap them in a struct or `std::tuple`.

Going back to our initial question ("why do we need in-out values?"), the reasons are that the
alternative is taking all inputs by value and return all outputs by value, which incurs the cost
of extra copies and is cumbersome to handle with multiple returns.

```cpp
BigObject foo(BigObject bo);  // Makes unnecessary copies of BigObject

std::tuple<B, C> bar(const A& a, B b, C c); // Return is painful to handle
```

But are they really?

### Cost of copies

Unnecessary copies can be done away through move semantics. For example, if you want
to apply a transformation on a BigObject, you want to pass it as in-out parameter to save the costly
copy. But post C++11 that's not a concern anymore:

```cpp
BigObject foo(BigObject bo);

BigObject bo;
bo = foo(std::move(bo)); // No copy here
```

Even better: this enable function `foo()` to now accept temporary values, while in the past callers
*had* to create a local object to pass as in-out, even if the only one they really wanted to store
was the output one.

```cpp
BigObject foo(BigObject bo);
void bar(BigObject& bar);

// What we intend is to store the result of foo() from an temporary BigObject
BigObject bo = foo(BigObject("hello world", 42));

// Do you think this variant is more expressive about what we're doing here?
BigObject bo("hello world", 42);
foo(bo);
```

The gain would be even more obvious if `foo()` was `constexpr` because the compiler could run
the transformation at compile time and only construct the result.

### Handling multiple result values

Storing the results is not that painful since C++11 with `std::tie()`:

```cpp
std::tuple<B, C> bar(const A& a, B b, C c);

B b("hello");
C c(3.14);
std::tie(b, c) = bar(A(42), std::move(b), std::move(c));
```

If caller is supplying temporary values, it's also not a problem since C++17:

```cpp
std::tuple<B, C> bar(const A& a, B b, C c);

auto [b, c] = bar(A(42), B("hello"), C(3.14));
```

The only issue comes when there's a need to mix temporaries and existing variables as input-output
arguments.
Since structured bindings can only declare new variables, you can't use it. And obviously
`std::tie()` can't create a new variable. So at best we would go back to variant #1 and refrain
from using temporaries.

## A few last words

I will concede I do not have significant personal feedback on that idea yet, this is mostly something
that I came up with while following the debate on Twitter.

I would be very interested in getting feedback from my readers. Did you try it? Did you encounter
limitations or use cases that it didn't solve?

So far the only exception I could think of are operators (streams for example) but that's already
a different syntax on its own so I'm not sure explicit references are needed there.

In any case, if you're interested give it a try and tell me about your experience!
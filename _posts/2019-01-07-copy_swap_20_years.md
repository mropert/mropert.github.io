---
layout: post
title: Copy and Swap, 20 years later
tags: [cpp]
description: >
  Copy and Swap is an elegant (if venerable) C++ idiom than I learnt to appreciate for quite some
  time without much afterthought. It's simple, clean and does the job. But is there a catch?
author: mropert
---

Once upon a time, in the 90s, we started preaching one of the oldest pillars of Modern C++
that is [RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization).
We taught programmers the simple rule that a constructor must leave an object in a usable state,
that we should able to copy it, and that the destructor must clean all owned resources, no matter
what.

To help reasoning, we explained the [Rule of Three](https://en.wikipedia.org/wiki/Rule_of_three_(C%2B%2B_programming))
which reminds everyone that if they customize either the copy constructor, assignment operator
or destructor, they should most likely also do something about the other two.

Then came C++11 and with move semantics it became the Rule of Five which is basically the same,
but adds the move constructor and move assignment operator to the list.

Things started to become confusing so nowadays we prefer to advertise the
[Rule of Zero](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-zero) which basically
says "when in doubt, use `= default`". When using STL containers and smart pointers, your compiler
will generate suitable defaults for most classes, even the ones that hold resources.

But why do we have all those rules in the first place?

## It's dangerous to go alone

Copy/move constructors, copy/move assignment operators and destructors are the key part
objects' lifecycle. If one is wrong, users will get dangling references, leaks, double `delete`s
and other unsavoury things. And of course they need to do that without leaking anything if an
exception occurs.

Since the best way to avoid writing bugs is to write no code at all, the Rule of Zero is obviously
an excellent solution. Except sometimes we cannot rely on it. In those cases (for example when
writing custom containers) we will need to follow the Rule of Five.

This comes with some constraints:
* Our five functions need to form a consistent whole when they acquire, copy, move and destroy
  resources, else we'll leak or leave dangling pointers
* Some operations like construction and assignment are quite similar so we would prefer to write one
  by calling the other (again reuse reduces the amount of code to review)
* Construction and memory allocation may throw, meaning we must offer at least
  [basic exception safety guarantee](https://en.wikipedia.org/wiki/Exception_safety).
* All those should be efficient, as there would be little reason to write custom containers that
  underperform the STL (why not use the STL in that case?).

Fortunately, the problem is not a recent one, so we have Copy and Swap.

## Back to the 90s

The oldest reference to this idiom I could find is from Herb Sutter's 
[Guru of the Week #59](http://gotw.ca/gotw/059.htm), published in July 1999. The phrasing of
the article seems to reference the pattern as already known, so I suspect it's even older.

The concept is well known:
1. Write a destructor that deletes any owned resource
2. Write a copy constructor that duplicates any owned resource and takes ownership of it
3. Write a non-throwing `swap()` function that will exchange the contents of two containers by
   swapping the internal bits
4. Write the copy-assignment operator by making a temporary copy of the source object, then swap the
   copy with `this`.

This 4th point is the most elegant and important part of the idiom. Copy-assignment is usually
the trickiest one to write since it must delete existing content, insert a copy of the source
objects and survive if an exception is thrown somewhere in the process.

We solve all that by writing this simple code that works whatever we did in the other 2 (or 4):
```cpp
T& operator=(const T& rhs)
{
  T tmp(rhs);
  swap(tmp);
  return *this;
}
```

The idiom even scales to the Rule of Five easily by throwing one `std::move` in the recipe:
```cpp
T& operator=(T&& rhs)
{
  T tmp(std::move(rhs));
  swap(tmp);
  return *this;
}
```

In 3 lines we solved the problem while offering strong exception guarantee, that's brilliant!
That part is more than 20 years old and I still find it magical.

Well, except one thing...

## The catch

Remember the constraints we enumerated before? Especially the one about performance? Because we
have a problem here. Two actually.

1. With Copy and Swap, we will *always* allocate new resources then throw the present ones away.
   Even if our collection could fit in the already allocate storage (remember it's an assignment).
   And as we know allocation can be unpredictably slow. Especially for small collections that would
   otherwise have been fairly cheap to copy.
2. The whole operation will at some point require 3 times the resources: one for `this`, one
   for the source object, and one for the intermediate copy. We do a three-way swap and the cost
   is one extra copy of a potentially very large collection. CPU wise we will still keep to one copy
   operation per element which is perfectly fine, but for memory this comes with an extra 50% cost.

Alas! But is there a better option?

There is, but it will not be free. Copy and Swap is not an old and outdated idiom to which we have
a much better answer today. It remains one of the best solutions to the problem. The reason is that
to offer strong exception guarantee, there is no way around it. There must be a temporary copy
done first that we can simply delete if something goes wrong without touching the existing
collection.

To get better performance, we will have to give up something.

## Warrantee voids if exception happens

Sadly we don't have many options, we have to demote the strong exception safety guarantee to simply
basic. Our object will still be destructible if an exception occur during assignment but the content
of our collection will be unspecified.

For example let's say we're making a `vector`-like container:

```cpp
MyVector& operator=(const MyVector& rhs)
{
  if (this != &rhs)
  {
    clear();              // Deletes content but leaves buffer itself intact
    reserve(rhs.size());  // Reallocates buffer if needed
    std::uninitialized_copy(rhs.begin(), rhs.end(), m_end);
    m_end += rhs.size();
  }
  return *this;
}
```

There, we do the same job, but if an exception is thrown we only offer basic guarantee (the vector
will be empty). In exchange we can reuse the same buffer if suitable and avoid a costly `new`.

Well almost. It would be preferable to ensure that `reserve()` is smart enough to require extra
runtime memory only when growing from a non-zero size.
On the other hand, the will probably be unhappy if a failed reallocation destroys all his data
when he calls `reserve()` so we still need to offer a strong guarantee:

```cpp
using storage_type = std::aligned_storage_t<sizeof(T), alignof(T)>;

void reserve(std::size_t new_capacity)
{
  if (new_capacity > capacity())
  {
    if (empty())
    {
      // We don't have any value to preserve, we can delete first
      delete[] reinterpret_cast<storage_type*>(m_begin);
      m_begin = reinterpret_cast<T*>(new storage_type[new_capacity]);
      m_end = m_begin;
      m_end_of_storage = m_begin + new_capacity;
    }
    else
    {
      // Alloc first, if OK move data and then delete
      auto new_storage = std::make_unique<storage_type[]>(new_capacity);
      auto new_begin = reinterpret_cast<T*>(new_storage.get());
      std::uninitialized_move(m_begin, m_end, new_begin);
      std::destroy(m_begin, m_end);
      delete[] reinterpret_cast<storage_type*>(m_begin);
      m_end = new_begin + size();
      m_begin = new_begin;     
      m_end_of_storage = m_begin + new_capacity;
      new_storage.release();
    }
  }
}
```

Since `reserve()` is a no-op when there is already enough space to fit, our copy assignment
operator will reuse the existing buffer. And if reallocation is needed, since we called `clear()`
first it will delete then allocate instead of the other way around to save up runtime memory.

## In Conclusion

20 years later, it feels like Copy and Swap still does what we expect of him: ease up the
implementation of the Rule of Three (or Five) while offering strong guarantee.

We can do better by dropping that guarantee to basic, but as always software engineering is a matter
of tradeoffs. We gave up something and increased the maintenance cost of our container (the code
is clearly harder to review and understand than the original Copy and Swap) to save up on precious
allocations in some cases. You may also have noticed that our first version was generic, was the
second one would require different implementation for containers.

Which choice did the C++ Standard by the way? Well, as far as I know, the
[standard doesn't say](https://en.cppreference.com/w/cpp/container/vector/operator%3D). There
is no exception guarantee specified for copy assignment so it would seem that implementers are
free to choose. A quick glance at MSVC shows me that they went for basic guarantee so I'd expect
Clang and GCC to be similar.

I'd like to give credit to the great Howard Hinnant who pops up every now and then on Stack
Overflow threads about Copy and Swap to remind us of the tradeoff. He was a good inspiration to this
article and you can find a whole talk about the matter
[here](https://www.youtube.com/watch?v=vLinb2fgkHk) in which he makes a case for always
having basic (and not strong) guarantee for assignments and then use a generic template function
to do a strong copy and swap when really needed.

Oh and by the way, did you notice that we wrote vector copy and reallocation without a single
raw loop using only the STL? :)
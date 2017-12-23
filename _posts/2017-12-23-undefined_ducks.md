---
layout: post
title: Follow-up to 'Better polymorphic ducks'
tags: [cpp]
description: >
  A quick follow-up on the previous article 'Better polymorphic ducks' to address two small issues.
author: mropert
---

Peer reviews are a great source of insight. This is why we do code reviews and talks rehearsals.
Blog posts are no exceptions and my astute readers pointed two things in the previous article
[Better polymorphic ducks](/2017/12/17/better_polymorphic_ducks/) that I will now share with you.

### Strict aliasing

Our implementation of Small Buffer Optimization did rely on undefined behaviour, because doing a `reinterpret_cast`
from derived to base is not guaranteed to do what you expect.

Let's look at the code again:

```cpp
  template <typename T>
  duck_t(T&& impl) {
    new (&m_storage) model_t<std::decay_t<T>>(std::forward<T>(impl));
  }

  inline concept_t& get() {
    return *reinterpret_cast<concept_t*>(&m_storage);
  }
```
Here we take the address of the storage (where we built a `model_t`) and reintrepret it as a `concept_t`. As far
as the C++ standard is concerned, this is illegal. The compiler is allowed to layout the two classes in a way that
they do not alias each other. `static_cast` will work around this by shifting addresses if needed, but `reinterpret_cast`
won't. I used to believe that the only difference between the two was the types of casts that were considered legal
but as it turns out there is more at play.

A solution to that is to save the result of the `new` expression as `concept_t*` and use it. Creating a pointer to base
from derived is an implicit cast that will adjust offsets if needed. The drawback is that we use an extra pointer
that costs both space and time (because of the extra indirection).

I've made a new bench and as you can see, it is indeed a bit slower:
[http://quick-bench.com/LaMiEbpTLDeyZWcUQIaL3frBzgk](http://quick-bench.com/LaMiEbpTLDeyZWcUQIaL3frBzgk).
I've also used that occasion to add some missing SFINAE.

Sean Parent himself ran into exactly the same error when demonstrating this technique at Meeting C++ this year.
I encourage you to read his analysis and suggestion to relax the standard on that matter:
[http://stlab.cc/tips/small-object-optimizations.html](http://stlab.cc/tips/small-object-optimizations.html).

### Metaclasses

In my article, I also suggested that we could possibly include this mechanism directly into the standard. One
of the arguments was that there was no other option so far, because compile-time code injection in the metaclasses
proposal did not go that far.

As Simon Brand pointed out, the latest draft does, in fact, allow it. You can take a look at his proof of concept right here:
[https://github.com/TartanLlama/typeclasses](https://github.com/TartanLlama/typeclasses).

I don't know about you, but now I wonder what else I can do with metaclasses :)

As always, here is the (so far) final code: [https://godbolt.org/g/RnhEUu](https://godbolt.org/g/RnhEUu).

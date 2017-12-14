---
layout: post
title: Better polymorphic ducks
tags: [cpp]
description: >
  In a previous post we discussed how to combine duck typing with runtime polymorphism. A polymorphic duck if you will.
  Today let's see how we can improve on our original design to remove some limitations.
author: mropert
---

The use of the TEPS (Type Erasure (Sean) Parent Style) we have shown in the [first part](/2017/11/30/polymorphic_ducks/)
of this series gave us what I call a Polymorphic Duck: something that can walk like a duck and quack like a duck,
but it not necessarly inherited from a base `Duck` class. Moreover, the `quack()` and `walk()` are expected
to be free function, not methods, which allows for looser coupling (especially if they take arguments that have
nothing to do with the duck itself).

```cpp
struct duck_t { /* magic */ };

class Swan { /* ... */ };
void walk(const Swan&);
void quack(const Swan &);

class Goose { /* ... */ };
void walk(const Goose& g);
void quack(const Goose& g);

std::vector<duck_t> bar(); // Might return a mix Swans, Geese, or even actual Ducks!

void foo() {
  for (const auto& duck : bar()) {
    walk(duck);
    quack(duck);
  }
}
```

Today the question is: how can we improve our duck?

If you answered "by pairing it with a red from southwestern France such as Cahors" you would be absolutely right, but this is not a
topic we will explore today. Instead we will look at plain and boring C++ code. If you remember my previous article we listed a
bunch of limitations that we would like to see gone:
* We can't copy our objects
* We store our objects outside the container through a `new`
* We need to write a bit of boilerplate for our polymorphic concept that we will probably copy/paste a lot around for each
  concept we have.

Let's try to handle each one in order.

## Polymorphic copy

The inability to make a polymorphic copy of an object through its inteface is a frequently heard complaint when you start dealing with
runtime polymorphism in C++. For example, all Java `Object` have a native `clone()` method that returns a full copy of the actual
implementation behind the interface, whatever it is. In C++ you can still do it by hand, but that's painful because you need to 
write an implementation for each derived class that usually looks like `return new T(*this)`.

Fortunately, our magical duck can generate that for all concrete implementations it will hold:

```cpp
class duck_t {
  struct concept_t {
    // ...

    // Addendum:
    virtual std::unique_ptr<concept_t> clone() const = 0;
  };

  template <typename T>
  struct model_t : public concept_t {
    // ...

    // Addendum:
    std::unique_ptr<concept_t> clone() const override {
      return std::unique_ptr<concept_t>(new model_t(*this));
    }
  };

  // ...

public:
  // Replaces the previously deleted ones:
  duck_t(const duck_t& d)
    : m_impl(d.m_impl->clone()) {}

  duck_t& operator=(const duck_t& d) {
    m_impl = d.m_impl.clone();
    return *this;
  }
};
```

All done! Now we can copy any duck that fits the concept at no additional cost. The only thing we require is the concrete
type to be copy-constructible.

```cpp
duck_t someDuck = Goose();
walk(someDuck); // The goose walks
someDuck = generateDuck(); // What could it really be? I don't know...
walk(someDuck); // It's ALIVE!

duck_t d(someDuck); // We can also copy construct one from the other
quack(d);
```

With that done, we can go on to the next issue...

## Storage

In the current prototype we have, the actual objects are stored outside the polymorphic wrapper, using a `new`.
This could be inefficient for a couple reasons:
* Constructing a `duck_t` on the stack will still do a heap allocation behind the scenes, which may impose a penalty depending on the domain.
  For example, most realtime and low-latency code avoid using the heap entirely because the alloc time is unpredictable.
* Putting all our Ducks in an array-like container will not guarantee the actual data ends up in the same memory region,
  decreasing locality of reference.

Let's try to replace the `unique_ptr` by a static buffer. Since we'll be changing a lot
of lines I'll show the complete source this time:

```cpp
class duck_t {
  struct concept_t {
    virtual ~concept_t() {}
    virtual void clone(void* addr) const = 0;
    virtual void move_clone(void *addr) = 0;
    // duck operations
    virtual void do_walk() const = 0;
    virtual void do_quack() const = 0;
  };

  struct empty_t : public concept_t { 
    virtual void clone(void* addr) const override {
      new (addr) empty_t(*this);
    }
    virtual void move_clone(void* addr) override {
      new (addr) empty_t(std::move(*this));
    }
    virtual void do_walk() const override {
      throw std::invalid_argument("Bad function call");
    }
    virtual void do_quack() const override {
      throw std::invalid_argument("Bad function call");
    }
  };

  template <typename T>
  struct model_t : public concept_t {
    model_t() = default;
    model_t(const T& v) : m_data(v) {}
    model_t(T&& v) : m_data(std::move(v)) {}

    virtual void clone(void* addr) const override {
      new (addr) model_t(*this);
    }
    virtual void move_clone(void* addr) override {
      new (addr) model_t(std::move(*this));
    }
    virtual void do_walk() const override {
      walk(m_data);
    }
    virtual void do_quack() const override {
      quack(m_data);
    }
    T m_data;
  };

  static constexpr std::size_t BufferSize = 64;
  std::aligned_storage_t<BufferSize> m_storage;

  inline concept_t& get() {
    return *reinterpret_cast<concept_t*>(&m_storage);
  }

  inline const concept_t& get() const {
    return *reinterpret_cast<const concept_t*>(&m_storage);
  }

public:
  duck_t() {
    new (&m_storage) empty_t;
  }

  duck_t(const duck_t& d) {
    d.get().clone(&m_storage);
  }

  duck_t(duck_t&& d) {
    d.get().clone(&m_storage);
  }

  template <typename T>
  duck_t(T&& impl) {
    static_assert(sizeof(T) <= BufferSize, "Object too big");
    new (&m_storage) model_t<std::decay_t<T>>(std::forward<T>(impl));
  }

 duck_t& operator=(const duck_t& d) {
   get().~concept_t();
   d.get().clone(&m_storage);
   return *this;
  }

  duck_t& operator=(duck_t&& d) {
    get().~concept_t();
    d.get().move_clone(&m_storage);
    return *this;
  }

  template <typename T>
  duck_t& operator=(T&& impl) {
    static_assert(sizeof(T) <= BufferSize, "Object too big");
    get().~concept_t();
    new (&m_storage) model_t<std::decay_t<T>>(std::forward<T>(impl));
    return *this;
  }

  friend void quack(const duck_t& d) { d.get().do_quack(); }
  friend void walk(const duck_t& d) { d.get().do_walk(); }
};
```

This way we remove all heap allocation and ensure our objects' data will be contiguous in memory if put in an array-style container.
As you can see in this [quick bench](http://quick-bench.com/vGN5ObZxa-thecR1cbbc04MS9M0), we are about 7 times faster on vector copy.

Access times aren't much faster though, as you can see [here](http://quick-bench.com/wdQedCOdcXkKdOqwZlVtLljNA2A). It is possible
that with better simulation of memory fragmentation and operations that rely more on the data from inside the ducks, we would
observe something. If one of my readers manages to do it, I'll be sure to mention it. 

If you are using GCC, then this version is much better also on the call case but only because apparently the original version
has terrible peformance: [http://quick-bench.com/9GSB69ZOF32zEU6tAj6wJjeZLIM](http://quick-bench.com/9GSB69ZOF32zEU6tAj6wJjeZLIM).
If you look at the numbers, you'll see that the baseline is 3.5x times slower on GCC than on Clang, but the second version
yields similar numbers.

The drawback of this version is that every object now takes 64 bytes, and objects bigger than this can't be handled.
There are two ways to improve this:
* We can fine tune the buffer size depending on what we expect to put there. If there's a big variation in the possible sizes,
  we are still going to waste a sensible chunk of memory.
* Transform our buffer in a real Small Buffer Optimization: if T is smaller than a certain size, we do store it in place, but if it's bigger
  than the treshold, we heap allocate instead. This is a bit more complex to write but it can be done. I will not elaborate further here
  because the topic would be more suited for a dedicated post.

## Removing some indirections

If we properly declare our `model_t` (and `empty_t`) as `final`, the compiler may be able to inline some of the virtual calls.

As you can see in this [benchmark](http://quick-bench.com/jurZeyi8BEZSPAs5ftHmzRD9dM8), we gain almost 2x improvement in the copy case,
which is pretty impressive for a single keyword.

Still the gain depends a lot on the test case as you can see in our [call bench](http://quick-bench.com/5jCdu9-XbBdpAYjHYL9aDhe7UO8),
which shows only minimal improvement. This time both GCC and Clang show comparable results.

## Generating the duck

The last big issue we have with this technique is the fact that we need to write 200 lines for each concept we want to use
this way, each time only really changing the `do_xxx()` methods and their final wrapper. It would be great if some tool
could handle that for us.

Unfortunately, I don't think this is possible with C++ today without an external tool (unless maybe we use macros heavily,
but who would want that? :)). The easiest way would be to model our concept by declaring the signatures we want in a class
or namespace and then use a [clang](http://clang.llvm.org/docs/IntroductionToTheClangAST.html) based program to read it
and generate what's needed in a .hpp file.

Will the future help? Presently, not really. Concepts will offer a way to define our Duck concept better but we'll still
need a clang-based tool to parse it and generate the wrappers. Metaclasses and static reflection do not provide code injection
so it wouldn't help either.

An idea that was suggested to me by Joel Falcou would be to simply integrate the pattern in the language.
Some kind of "virtual" concept. For example we would write something like this:

```cpp
template <typename T>
virtual concept duck_t = requires(T d) {
   quack(d);
   walk(d);
};
```

And have the compiler generate all we need to wrap any object that satisfies the concept described.
I will leave you to that final thought, don't hesitate to tell me what you think about it.

As always, you can find the final code is here: [https://godbolt.org/g/kFp23k](https://godbolt.org/g/kFp23k).
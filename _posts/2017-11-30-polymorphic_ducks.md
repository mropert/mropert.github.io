---
layout: post
title: Polymorphic ducks
tags: [cpp]
description: >
  It's no secret C++ favours static polymorphism. But sometimes, runtime polymorphism is needed and suddenly we find ourselves down
  the virtual rabbit hole. Do not despair, for there are ways to avoid this madness.
author: mropert
customjs:
 - https://platform.twitter.com/widgets.js
---

I hate the `virtual` keyword. Inheritance fills me with a sense of dread. I can always quote half a dozen technical reasons to explain it.
People much more smarter than me have discussed why it's the worst form of composition.

But deep down I know the real reason is that when I was first offered to use C++ instead of C,
I made a nice UML class diagram and then proceeded to write half a dozen class in hierarchy
that had about one concrete implementation (and probably still does). We all have our crosses to bear.

Today, as much as I try to avoid dynamic polymorphism, it is sometimes the right tool for the job. When you have no way of knowing which 
concrete implementation of a concept will be used at compile time, or when you want your business logic to be testable in isolation
without putting everything inside templates, you end up using it.

The problem is not as much with the paradigm but with the implementation. We like to pay for what we use, and while we only asked
for runtime polymorphism (and paid the cost of the vcall), we also got tighter coupling and lost regular typing as a bonus. Which
is terrible.

## The problem with traditional inheritance

Consider the following:
```cpp
class Drawable {
public:
   virtual ~Drawable() {}
   virtual void draw(Display& display) const = 0;
};
```

Here we have a bunch of objects we want to draw. We can't know at runtime which ones we will have, only that at some point
we will go through a list of them and draw them on a given `Display` with a call to `draw()`. This is all we care about. But we got
a lot more for our trouble:

All of our drawable objects must inherit `Drawable` and implement `draw()`, which means each of them has to know how to draw itself
(or save, or load, or compute or...) which is a clear violation of the principle of separation of concerns<sup>1</sup>. I could stop there
in my list of problems, we already reached the point of no return by coupling our implementation much more that it
reasonably should.

But it goes on:
* We can't copy-construct or copy-assign our objects anymore, at best we can implement a `clone()` virtual method everywhere.
* We can't have a list (or array or set or...) of our objects, we need a list of `Drawable*` (or a `std::unique_ptr<Drawable>`).
  It prevents us from using most standard algorithms which expect values and not pointers.
* We can't store them by value, an owning container will have to `new` each one instead of putting them in a contiguous buffer
  or on the stack, which is terrible for modern CPUs caches.
* Our code will get cluttered with calls to `new` or `std::make_unique` everytime we want to create a drawable.

## Alternatives

In his [talk at CppCon 2017](https://www.youtube.com/watch?v=gVGtNFg4ay0), Louis Dionne showed a bunch of alternative
techniques around that problem which I found interesting but ultimately too complex to present here<sup>2</sup>.

Instead, I propose we simply use the time tested technique of WWSPD: What Would Sean Parent Do?

If you answered `std::rotate()`, nice try but unfortunately it was not the right answer this time. No, the right answer was
presented in his talk: [Better Code: Runtime Polymorphism](https://www.youtube.com/watch?v=QGcVXgEVMJg). Here's how it works:

### Erasing the type

First we take the concept we want to use and make it an interface like we would have done in a naive way:
```cpp
struct concept_t {
   virtual ~concept_t() {}
   virtual void draw() const = 0;
};
```
You may feel cheated after what I said in the previous section but bear with me. Next we create a templated implementation:
```cpp
template <typename T>
struct model_t : public concept_t {
  model_t() = default;
  model_t(const T& v) : m_data(v) {}
  model_t(T&& v) : m_data(std::move(v)) {}
  
  void draw() const { m_data.draw(); }
  
  T m_data;
};
```
We are starting to get there but something is still missing... now we complete the type erasure by wrapping all that in class:
```cpp
class drawable {
  struct concept_t { /* ... */ };
  template <typename T> struct model_t : public concept_t { /* ... */ };
public:
  drawable() = default;

  template <typename T>
  drawable(T&& impl) : m_impl(new model_t<T>(std::forward<T>(impl))) {}

  template <typename T>
  drawable& operator=(T&& impl) {
    m_impl.reset(new model_t<T>(std::forward<T>(impl)));
    return *this;
  }
   
  void draw() const { m_impl->draw(); }
   
private:
  std::unique_ptr<concept_t> m_impl;
};
```

Finally we can manipulate our objects as polymorphic regular objects:
```cpp
std::vector<drawable> objects;
objects.push_back(Rectangle(12, 42));
objects.push_back(Circle(10));
objects.push_back(Sprite("assets/monster.png"));

for (const auto& o : objects)
   o.draw();
```

### Getting rid of member functions

That's better, but we still haven't fulfilled our most important requirement: decouple objects from their drawing implementation.
Fortunately, it's quite easy now. We just need to replace a method call by a function call:
```cpp
class drawable {
  struct concept_t {
    virtual ~concept_t() {}
    virtual void do_draw() const = 0;
  };
  template <typename T>
  struct model_t : public concept_t {
    model_t() = default;
    model_t(const T& v) : m_data(v) {}
    model_t(T&& v) : m_data(std::move(v)) {}

    void do_draw() const override { draw(m_data); }

    T m_data;
  };
public:
  drawable() = default;

  template <typename T>
  drawable(T&& impl) : m_impl(new model_t<T>(std::forward<T>(impl))) {}

  template <typename T>
  drawable& operator=(T&& impl) {
    m_impl.reset(new model_t<T>(std::forward<T>(impl)));
    return *this;
  }
  
  friend void draw(const drawable& d) { d.m_impl->do_draw(); }

private:
  std::unique_ptr<concept_t> m_impl;
};
```
We just let the [ADL](http://en.cppreference.com/w/cpp/language/adl) do its magic. As long as the compiler can find a
`draw(const T&)` function, it will work fine. Next we can think about adding arguments (don't you feel like maybe we should
tell the program _where_ to draw?).

Our final calling code may now looks like this:
```cpp
for (const auto& o : objects)
   draw(o, display);
```

### Wrapping-up

We saw how to break the dependency between our objects representations and the functions supposed to use them. We achieved
runtime polymorphism without compromising our design. But we still come short of a few promises:

* We still can't copy our objects
* We still store our objects outside the container through a `new`
* We needed to write a bit of boilerplate for our polymorphic concept that we will probably copy/paste a lot around for each
  concept we have.

In a later post, we will see if and how we can solve those issues. But first we must address a much more important question:
how should we call this technique? Fortunately, I asked and Twitter answered:

<blockquote class="twitter-tweet" data-lang="en">
<p lang="en" dir="ltr">
<a href="https://twitter.com/SeanParent?ref_src=twsrc%5Etfw">@SeanParent</a>
approved (that&#39;s what liking on Twitter means, right?).
<br>So it is decided, cpp_acronyms.insert(&quot;TEPS&quot;, &quot;Type Erasure Parent Style&quot;);
<a href="https://t.co/h2Pu2bIphm">https://t.co/h2Pu2bIphm</a></p> &mdash; Mathieu Ropert (@MatRopert)
<a href="https://twitter.com/MatRopert/status/936362895000076288?ref_src=twsrc%5Etfw">November 30, 2017</a>
</blockquote>

So next time you see inheritance and `virtual` to achieve runtime polymorphism, think TEPS! (Thanks to Simon Brand for the idea :))

You can find the full final source here: [https://godbolt.org/g/iXr1Xj](https://godbolt.org/g/iXr1Xj).


<sup>1</sup> The traditional solution around this is the [Visitor](https://en.wikipedia.org/wiki/Visitor_pattern), which comes
at the cost of more virtual dispatch and more inheritance.

<sup>2</sup> In Louis' defense, he actually shown Sean's technique in his original talk but decided to cut it for CppCon
following some feedback he got in previous sessions (viewers found it too complex to follow).
I'm still puzzled by this because I find it so simple and elegant, different strokes I guess...

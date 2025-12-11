---
layout: post
title: "What makes a game tick? Part 8 - Data Driven Multi-Threading Implementation"
tags: [cpp, gamedev]
description: > 
  Let's talk about game simulations. Today we dive into the nitty-gritty bits of implementing
  data driven multi-threading.
author: mropert
---

[In our last episode in this series](/2025/10/06/making_games_tick_part7/) we presented the concept of task-based parallelism
with scheduling driven by data accesses. I recommend going back to it for a quick reminder because today we are gonna
talk about implementation. Let's get coding!

## Access denied

To be able to implement data-driven task parallelism, we need to establish two rules:
1. Tasks must declare what data they read and write
2. Data accesses within tasks must go through some middle man that will ensure they are only accessing data they declared

This may sound obvious, but this has some pretty important implications. First of all, no pointers. Or references.
Unique pointers and containers are OK as long as no other objects is allowed to keep a pointer to them.

Here are some examples:

```cpp
struct Army
{
    // Direct data members, all fine
    float health;
    float morale;
    float attack;
    float defense;
    
    Country* controller; // Bad
    const Province* location; // Still bad
    const Country& owner;  // Also bad

    std::vector<Equipment> equipment;  // Collection of direct members, fine
    std::unique_ptr<Emblem> emblem; // Fine as long as only accessed through the Army
};
```

So if we cannot have pointers, what can we do? Obviously we can't just declare that every object should not have
relationship to any other object, but we have to express those relationships in a way that does not allow for
unchecked pointer following.

## A pointer wrapper

A simple solution is to make a thin wrapper around pointers:

```cpp
template <typename T>
class obj_ptr {
public:
    constexpr obj_ptr() = default;
    constexpr explicit obj_ptr(T* obj) : m_ptr(obj) {}
    constexpr obj_ptr& operator=(T* obj)
    {
        m_ptr = obj;
        return *this;
    }

    constexpr void clear() { m_ptr = nullptr; }
    constexpr explicit operator bool() const { return m_ptr != nullptr; }
    constexpr auto operator<=>(const obj_ptr&) const = default;

private:
    template<typename... Types>
    friend class accessor;

    T* m_ptr = nullptr;
};
```

This `obj_ptr` wraps a pointer and offers similar logic except it's missing the actual de-reference operations.
As such it is safe to use because no one outside of the friend class `accessor` can access the `m_ptr` member.
Then the next step is to actually implement this `accessor` class that will act as our de-reference proxy.

```cpp
// Is a type allowed read-only access by a given accessor?
template<typename T, typename... AllowedTypes>
concept ro_access = details::contains_type<std::add_const_t<T>, AllowedTypes...>;

// Is a type allowed read-write access by a given accessor?
template<typename T, typename... AllowedTypes>
concept rw_access = details::contains_type<T, AllowedTypes...>;

template<typename... Types>
class accessor
{
public:
    constexpr accessor(const accessor&) = default;
    constexpr accessor(accessor&&) noexcept = default;

    template <rw_access<Types...> T>
    T* get(obj_ptr<T> ref)
    {
        return ref.m_ptr;
    }

    template <ro_access<Types...> T>
    const T* get(obj_ptr<T> ref)
    {
        return ref.m_ptr;
    }
private:
    constexpr accessor() = default;
    friend class task_executor;
};
```

Now, how does this work? The concepts `ro_access` and `rw_access` acts as a barrier, only emitting a `get()` functions
for a given `T` if that type is part of `Types`, and with the right `const` qualifier on the returned pointer. For example:

```cpp
void UpdateProvince(accessor<Province, Army, const CountryDiplomacy> access, obj_ptr<Province> p)
{
    // Ok, Province is part of accessor's types
    Province* province = access.get( p );
    for ( obj_ptr<Army> a : province->m_armies )
    {
        // Also ok, access.get() returns an `Army*` that is downcast to `const Army*`
        const Army* army_on_province = access.get( a );
        // Compute something ...
    }
    // Compile error: only have const access to diplomacy
    CountryDiplomacy* owner_diplo = access.get( province->m_owner_diplo ); 
    // Compile error: no access to navies
    const Navy* navy_in_port = access.get( province->m_navy_in_port ); 
}
```

As you can see, this way we ensure a given task (like `UpdateProvinces()`) only accesses what it promised it would,
in the way it promised it would (read or write) or else the task will fail to compile. With that guarantee in mind,
we can now check that two tasks are compatible to run in parallel, and we can even do it at compile time. All
it takes is extracting the type list from the first argument's signature, and check if any non const type in one
appears on the other.

## A viral pattern

One important consequence of using this kind of technique is that by necessity every function that a task may call now
has to add an accessor parameter to its signature. We should of course add a constructor to `accessor` that allows
for creating subs-accessors with either less types or types demoted from non-const to const. Still, it is a thing
that will annoy programmers and designers alike when iterating over features and that should be kept in mind.

Especially because one of the effect of such a viral pattern is that when one find themselves needing an extra data accessor
down in a leaf function, they need to edit the signature of _all_ callers recursively to add the required types.
On the other hand, I found that this encourages (forces?) programmers to see the performance of a given gameplay change,
because by its nature it forces them to bubble that change all the way up to the task declaration. It will
also make it quite blatant to the pull request reviewer when a new read/write access to common type is added.

Speaking of, the other advantage of this technique is that it makes it very obvious which game objects are
used all the time and would be potential candidates for being split, as we shown in the previous chapter.
Of course this does not replace a profiler, sometimes a type is only shared by a few tasks, but they are all
very expensive to run and should definitely be parallelized instead of run serially. You should always be
profiling your tick.

## Going further

There's of course more to be implemented here. We haven't written the scheduler, nor talked about how we should
handle the game object's primary storage. We will discuss part of those next time, although the specifics
of how to implement a full scheduler might be better done as part a github repository (no promises though).

Finally, I'd like to remind readers that this is a per _type_ access control solution, not a per _object_.
It is still possible to subdivide the work by turning a loop into a parallel variant, but that may warrant
more discussions also. Until then!

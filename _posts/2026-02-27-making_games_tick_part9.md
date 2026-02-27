---
layout: post
title: "What makes a game tick? Part 9 - Data Driven Multi-Threading Scheduler"
tags: [cpp, gamedev]
description: > 
  Let's talk about game simulations. Now that we have described he basics of data driven
  multi-threaded ticks, we look at how to build a thread safe scheduler for our tasks.
author: mropert
---

Back in [late 2025](/2025/12/11/making_games_tick_part8/) we started implementing data-driven multi
threaded ticks by making all game object lookups and dereferences go through a thin accessor. This in turn
forced us to describe which types a given tick task would need to read and write. And with that information,
we have everything we need to build a parallel scheduler.

## Task metadata

If you remember from [part 7](/2025/10/06/making_games_tick_part7/) we had described a set of tasks
that constituted our example tick and built a simple data access table.
I'll repeat it here for easy reference:

| Task            | Economy | Diplomacy | Modifiers | Provinces | Armies | Navies | AI |
|-----------------|---------|-----------|-----------|-----------|--------|--------|----|
| UpdateModifiers |         |           | 🖊️       |            |        |        |    |
| UpdateProvinces |         | 📖        |          | 🖊️        | 📖    |        |    |
| UpdateEconomy   | 🖊️      |           |          | 📖        |        |        |    |
| UpdateDiplomacy |         | 🖊️        |          | 📖        |        |        |    |
| UpdateArmies    |         | 📖        | 📖       | 📖        | 🖊️    |        |    |
| UpdateNavies    |         | 📖        | 📖       | 📖        |       | 🖊️     |    |
| UpdateAI        | 📖      | 📖       | 📖       | 📖        | 📖    | 📖     | 🖊️ |

First, let's translate those into C++ function signatures using the `accessor` type we described before:

```cpp
namespace tick_tasks
{
void UpdateModifiers(accessor<Modifiers>);
void UpdateProvinces(accessor<const Army, const CountryDiplomacy, Province>);
void UpdateEconomy(accessor<const Province, CountryEconomy>);
void UpdateDiplomacy(accessor<const Province, CountryDiplomacy>);
void UpdateArmies(accessor<const CountryDiplomacy, const Modifiers, const Province, Army>);
void UpdateNavies(accessor<const CountryDiplomacy, const Modifiers, const Province, Navy>);
void UpdateAI(accessor<const Army,
                       const CountryDiplomacy,
                       const CountryEconomy,
                       const Modifiers,
                       const Province,
                       const Navy,
                       CountryAI>);
}
```

As you can see, the data accesses of each task are part of the function signature (through the type of their first argument).
With some simple template meta programming we can access it. The obvious place to capture it would be through whatever
registry mechanism we use to tell our scheduler which tasks are part of the tick.

```cpp
class scheduler
{
public:
    template<typename... Types>
    void register_task(void (*task)(accessor<Types...>))
    {
        _tasks.emplace_back(task);
    }

private:
    std::vector<task> _tasks;
};

namespace tick_task
{
void RegisterTasks(scheduler& sched)
{
    sched.register_task(UpdateModifiers);
    sched.register_task(UpdateProvinces);
    sched.register_task(UpdateEconomy);
    // ...
}
}
```

## Task polymorphism

To implement our task storage we will need some form of type erasure to turn our registry into a `vector` of `task` objects
that we can then manipulate in a more traditional fashion.
While I enjoy template metaprogramming, I find it simpler to keep things on the low side as soon as
it looks like we'll need do any kind of iteration or sorting. Some programming languages are really good
at making types manipulation easy, but C++ isn't one of them (we can revisit this assertion once C++26 reflection is available).

This means that some parts of our scheduler will do things at runtime that could possibly be done at compile time
(such as building a task dependency graph), but after experimenting with both options I found the runtime version
much simpler to use and maintain, and the cost of building the task graph one time at startup was negligible.

So, let's implement the task type erasure.

```cpp
namespace details
{
template<typename T>
void add_type(std::flat_set<std::size_t>& reads, std::flat_set<std::size_t>& writes)
{
    if constexpr(std::is_const_v<T>)
    {
        reads.emplace(typeid(std::remove_const_t<T>).hash_code());
    }
    else
    {
        writes.emplace(typeid(T).hash_code());
    }
}
template <typename Tuple, std::size_t... I>
void add_types_from_tuple(std::flat_set<std::size_t>& reads,
                          std::flat_set<std::size_t>& writes,
                          std::index_sequence<I...>)
{
    // Might be able to skip the need for using tuple with C++26 pack indexing?
    (add_type<typename std::tuple_element<I, Tuple>::type>(reads, writes), ...);
}
}

class task
{
public:
    template<typename... Types>
    task(void (*task_fn)(accessor<Types...>))
        : _fn([task_fn](Gamestate& gamestate)
              {
                  auto accessor = gamestate.make_accessor<Types...>();
                  task_fn(accessor);
              }
          )
    {
        using Tuple = std::tuple<Types...>;
        details::add_types_from_tuple<Tuple>(
            _reads,
            _writes,
            std::make_index_sequence<std::tuple_size_v<Tuple>>{});
    }

    void run(Gamestate& gamestate) const { _fn(gamestate); }
    const auto& get_reads() const { return _reads; }
    const auto& get_writes() const { return _writes; }

private:
    std::function<void(Gamestate&)> _fn;
    std::set<size_t> _reads;
    std::set<size_t> _writes;
};
```

And with that we should have our basic task. The idea behind the interface is that all tasks can be run
as a callable that takes a reference to the `Gamestate`, and provide a set of which types are
being read or written by the task. Assuming the task is only called from a sane environment
(like a task graph built upon our constraints), this is potential place for where we can create
an `accessor`. In general you want to make sure gamestate accessors are created at safe points,
but the nice thing about this pattern is that those become the _only_ points where you can possibly
create a data race. Anywhere else would trigger a compile error.

In this example we use the hash code from the `typeid` which allows us to work with any given type.
Alternatively we could have our own registry of allowed types which assigns an index to each registered
type and use a bitset instead, it would be more intrusive as we need to explicitly register types,
but it would simplify some the graph construction because finding intersection between 2 bitsets
is a simple binary AND.

## The scheduler itself

Once we have turned all our tasks into a vector, it becomes easier for us to create a task graph.
Here's a very basic implementation:

```cpp
// Utility to keep the rest readable
template<typename T>
bool intersects(const std::set<T>& s1, const std::set<T>& s2)
{
    std::vector<T> intersection;
    std::set_intersection(
        begin(s1), end(s1), begin(s2), end(s2), std::back_inserter(intersection));
    return !intersection.empty();
}

void build_graph(std::span<task> tasks)
{
    for (int i = 0; i < tasks.size(); ++i)
    {
        for (int j = 0; j < i - 1; ++j)
        {
            if (intersects(tasks[i].get_reads(), tasks[j].get_writes())
                || intersects(tasks[i].get_writes(), tasks[j].get_reads())
                || intersects(tasks[i].get_writes(), tasks[j].get_writes()))
            {
                tasks[j].add_dependency(task[i]);
            }
        }
    }
}
```

And with that, we have a built a task graph that we can then feed to our favourite threading library
to run in parallel.

Next time, we wil look at data storage and how to tie all this together. See you there!

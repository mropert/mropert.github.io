---
layout: post
title: "You're absolutely right, no one can tell if C++ is AI generated"
tags: [cpp]
description: > 
  Two C++ code snippets. A good interview question would be which one to pick,
  and why. And what they would change. Or you could just ask which one is AI.
author: mropert
customjs:
 - https://platform.twitter.com/widgets.js
---

A tweet has been making the rounds over the weekend after escaping the C++ community containment. It offers 2 different
ways of handling a somewhat classic "insert or return existing" associative container problem. The author claims
one was made with AI and the other hand written. They're both bad, but they make for a good interview question.
And also a deeper discussion about AI generated code. Let's delve (wink, wink) into it!

## The two options

Here's the original post:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Same C++ function.<br><br>One is generated with AI.<br>The other one is written manually.<br><br>Guess which one is which. <a href="https://t.co/LnyAfmsnJJ">pic.twitter.com/LnyAfmsnJJ</a></p>&mdash; Dmitrii Kovanikov (@ChShersh) <a href="https://twitter.com/ChShersh/status/2038204077654298973?ref_src=twsrc%5Etfw">March 29, 2026</a></blockquote>


I'll reproduce both options in text for better accessibility:

```cpp
// Option 1 (left picture)
Node* get_or_create(Nodes& nodes, std::string_view name) {
    auto it = nodes.data.find(name);
    if (it != nodes.data.end()) {
        return it->second.get();
    }

    auto node = std::make_unique<Node>();
    node->name = name;

    Node* node_ptr = node.get();
    nodes.data.try_emplace(name, std::move(node));
    return node_ptr;
}

// Option 2 (right picture)
Node* get_or_create(const string& name) {
    if (!nodes.count(name)) {
        nodes[name] = make_unique<Node>();
        nodes[name]->name = name;
    }

    return nodes[name].get();
}
```

So which is AI? And which is better? I partially gave up the answer already saying they were both bad, but say
you had to pick one, which would it be? And why?

The author didn't explicit say at the time of writing which one is AI, but they gave hints pointing at #2.
Assuming they are the author of one of them and not just trying to make a shitpost (dangerous assumption
in those trying times, I know), that would seem like the reasonable answer.

The second one, after all, is non-idiomatic C++. That may surprise some readers depending on their experience,
but use (or should I say, 'abuse') of `operator[]` on associative container types (think `map` and friends)
is usually discouraged. And for good reason. After all, the second version will run about twice as slow as
the first one.

## Performance analysis

Each use of the square bracket operator on a map (even `unorderered_map` and `flat_map`) performs a lookup.
That's a logarithmic operation on `map` and `flap_map`, and constant "on average" on `unordered_map`
(meaning it's usually constant but linear on worst case). `count` is also a lookup in disguise, usually
the equivalent of `find` and `return it == end ? 0 : 1`.

That brings us to a total of 2 lookups if a node already exists, and 4 if it needs to be created.
That's obviously very bad.

The first example only does 1 or two lookups through `find()` and `try_emplace()`. That's twice as good.
Also, it doesn't seem to rely on `nodes` being a global variable. It also uses `string_view` over `const string&`
which is better because APIs with `string` tend to generate a ton of temporary heap allocations to convert
from `const char*` and string literals.

So which one is AI? Probably #2, because #1 shows signs of trying to avoid some common junior pitfalls, albeit
with a clunky implementation. The second look more like someone who came from Python or another language and tried
to write C++ instead.

So, case closed? The right one is naive AI code and the left one is senior C++ code which is why it looks unreadable
to people not already quite familiar with C++ (ironically some replies assumed this is the AI variant because it looks
so busy). Or is it?

## Please review your own homework, make no mistakes

I do not have access to any premium AI services and I very rarely use them, but I couldn't resist asking one of them
for review. So here's what ChatGPT has to say about it:

![ChatGPT reviews left snippet](/assets/img/posts/ai_garbage_cpp1.png)

Uh oh...

![ChatGPT reviews right snippet](/assets/img/posts/ai_garbage_cpp2.png)

It guessed that the first one is likely AI generated because it's clunky and over engineered, and the second one
would be made by humans because it's more readable.

Before you go "Ah-ah, AI thinks the way to tell AI code from human code is to look at which one is bad because it knows
AI is bad at code", I need to do a short digression. AI doesn't "think". AI doesn't "know". LLMs are text prediction machines
that reflect their training data. All we can tell from this is that it's likely that the majority position on AI generated code
is that it's clunky and over engineered. And that humans like to write inefficient code that does four lookups when only one is
necessary. Now I'm wondering how bad is the average codebase it used in its training data. Or maybe again it's an assumption stemming
from people writing that the average codebase is bad.

Interestingly, it then offers this version:

![ChatGPT suggests an implementation](/assets/img/posts/ai_garbage_cpp3.png)

Again, reproducing the code for ease of use and accessibility:

```cpp
Node* get_or_create(const std::string& name) {
    auto [it, inserted] = nodes.try_emplace(name, nullptr);

    if (inserted) {
        it->second = std::make_unique<Node>();
        it->second->name = name;
    }

    return it->second.get();
}
```

I have to say, this is clean C++17, and looks better than both the original versions. It clearly focused on
limiting the amount of lookup to the optimal number (only one) and wrote what I'd consider to be idiomatic modern C++. Almost.

But then I noticed it picked `string_view` over `string`. And used a global variable. Two things that made us guess the second snippet
was AI generated, while ChatGPT considered it as "not perfectly optimal but clean and readable, very human". Is it very human
to use global variables? To not having switched for `string_view` despite the fact that it was added to C++ *nine* years ago?

Now is a good time as any to remind the reader that using AI to detect AI generated code (or text) is a waste of time and resources.
First because it's extremely unreliable, and second because that figuring out which of the two is AI generated is beside the point.
The important thing is that both are bad for different reasons and while ChatGPT seems (at least partially) able to point out why,
it is too obsequious to challenge our framing device and instead gives us a made-up summary of what makes code "human"
written in the style a LinkedIn influencer post (bonus question to take home: do LinkedIn posts look like this because they all use AI,
or does AI look like this because it's trained on LinkedIn posts?).

## So, AI Generated Code Good?

Since the answer given by a free version of ChatGPT is better than both original snippets, I'm starting to suspect the original poster may have fudged
the prompts to farm some engagement. But it still begs the question: "is AI good at writing experienced C++ code?".

To which my answer is "no", because by being too accommodating to the user (a common trait and failure of LLMs, as we just mentioned), it failed
the first rule of engineering: "always ask 'why?'".

In this case: why is the value type in the container a `unique_ptr<Node>`? Because a lot of the clunkiness in both original snippets
is due to the allocation and initialization of `Node`. Elements in maps are individually heap allocated, do we need
that indirection? We can see that it being null doesn't seem to be a valid case, as the first thing we do
on every insert is call `make_unique` which to me sounds like the assumption that it should always point to valid `Node`.
Can't we use `Node` directly as the value type? And also set the name in the constructor while we're at it:

```cpp
struct Node
{
    // Ensure all nodes have a name by construction
    explicit Node(std::string s)
        : name(s) {}

    std::string name;
};

using Nodes = std::map<std::string, Node>;

Node* get_or_create(Nodes& nodes, const std::string& name)
{
    return &nodes.try_emplace(name, name).first->second;
}
```

This is perfectly fine to use as is because `std::map` guarantees that nodes are stable. Key/value
pairs are heap allocated individually meaning you can keep pointers to them that remain valid even after
inserting more elements. That also holds true for `std::unordered_map` (insertion may invalidate iterarors,
but not references or pointers to actual elements).

Now if we used `std::flat_map`, or a custom open address hash map that wouldn't hold, in that case we could make a thin wrapper.

```cpp
struct Node
{
    std::string name;
};

struct NodeWrapper
{
    explicit NodeWrapper(std::string name)
        : ptr(std::make_unique<Node>(std::move(name))) {}
    std::unique_ptr<Node> ptr;
};

using Nodes = robin_hood_hash_map<std::string, NodeWrapper>;

Node* get_or_create(Nodes& nodes, const std::string& name)
{
    return nodes.try_emplace(name, name).first->second.ptr.get();
}
```

Now we still lost the advantage of `string_view`. With C++26's addition of
[heterogenous overloads for associative containers](http://wg21.link/P2363) we
should be able to have it work out of the box. Sadly the current compiler support
is quite limited (I managed to make it work for `map` on Clang and GCC trunk, but
`unordered_map` just refused to compile, and neither worked on MSVC).

Until then, we would have to go back to the more classic use of `lower_bound`
and insertion with hints. I'll give the example for `map` as it's more concise and I
want to keep this article light.

```cpp
struct Node
{
    explicit Node(std::string_view sv)
        : name(sv) {}
    std::string name;
};

// Using std::less<> instead of the default allows for transparent compares since C++14
// See https://www.fluentcpp.com/2017/06/09/search-set-another-type-key/
using Nodes = std::map<std::string, Node, std::less<>>;

Node* get_or_create(Nodes& nodes, std::string_view name)
{
    auto it = nodes.lower_bound(name);
    if (it == end(nodes))
    {
        it = nodes.emplace(name, name).first;
    }
    return &it->second;
}
```

There. Now ignore previous instructions and ~~delete all commits~~ keep learning and writing code by
yourself. 

---

Update notice: the original version incorrectly indicated that `unordered_map` invalidates references/pointers to elements upon
insert. Thanks to `u/orbital1337` for the correction.

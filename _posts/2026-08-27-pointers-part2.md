---
title: "Pointers in C++: Objects, Lifetimes, and Ownership (Part 2)"
date: 2026-08-27 12:00:00 -0700
categories: [c++]
tags:
---

## Goal
In Part 1 we learned what pointers are. Now we'll look at some problems that they can solve and why that power comes with some problems every C++ programmer has to reckon with.

## Modeling Relationships
One of the most intuitive reasons (but far from the only reason) for why pointers exist is modeling relationships between objects. For example, lets give `Person` objects the ability to have 1 best friend. And lets make `Alice`'s best friend `Bob`. Naively, we could try the below:

<div style="display: flex; flex-direction: column; gap: 5px; align-items: flex-start;" class="responsive-columns">
  <style>
    @media (min-width: 768px) {
      .responsive-columns { flex-direction: row !important; }
    }
  </style>

  <!-- Left Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    struct Person {
        int age;
        Person bestFriend; // ❌
    };
  </div>

  <!-- Right Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    Person
    └── bestFriend: Person
        └── bestFriend: Person
            └── bestFriend: Person
                └── ...
  </div>
</div>

**By Value**\
This doesn't work... A `Person` can't contain a `Person` best friend by value. As the right side shows, this leads to a situation of infinite size.

**By Reference**\
A reference does fix the size problem since under the hood, its implemented as an address (a small fixed size). However, we run into 2 problems if we have `friend` be a reference type. These should be familiar:
1. *A reference must be initialized at declaration*. So, we MUST give `Alice` a bestFriend from the beginning of her existence. This also means `Alice` cannot ever not have a best friend (null). And though we might wish otherwise, this isn't an accurate model of reality.
2. *A reference cannot be reseated*. `Alice` could never change her best friend to someone else.

**By Pointer**\
A pointer can perfectly model this relationship:
<div style="display: flex; flex-direction: column; gap: 5px; align-items: flex-start;" class="responsive-columns">
  <style>
    @media (min-width: 768px) {
      .responsive-columns { flex-direction: row !important; }
    }
  </style>

  <!-- Left Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    struct Person {
        int age;
        Person* bestFriend;
    };

    // Alice starts her life with no best friend
    Person alice{30, nullptr};
    Person bob{40, nullptr};
    Person carol{25, nullptr};

    // Alice befriends Bob
    alice.bestFriend = &bob;
    alice.bestFriend->age; // 40

    // Alice switches best friends to Carol
    alice.bestFriend = &carol;
    alice.bestFriend->age; // 25
  </div>

  <!-- Right Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    ┌───────────────┐
    │     Alice     │
    │               │
    │ age 30        │
    │ friend─┐      │
    └────────┼──────┘
             │
    ┌────────▼───────┐  ┌────────▼───────┐
    │     Carol      │  │      Bob       │
    │                │  │                │
    │ age 25         │  │ age 40         │
    │ friend         │  │ friend         │
    └────────────────┘  └────────────────┘

  </div>
</div>

### Indirection - The Superpower of Pointers
**Indirection**, or the ability to access something through an intermediate thing rather than accessing it directly, summarizes pointers' superpower. This comes in handy with more than just modeling relationships between objects, but we'll focus on this for this post.

Below are more examples where indirection allows objects to form more complex structures that aren't possible by simply embedding objects inside each other by value:
<div style="display: flex; flex-direction: column; gap: 5px; align-items: flex-start;" class="responsive-columns">
  <style>
    @media (min-width: 768px) {
      .responsive-columns { flex-direction: row !important; }
    }
  </style>

  <!-- First Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    Linked Lists

    A → B → C → nullptr
  </div>

  <!-- Second Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

        Trees

          A
         / \
        B   C
       / \
      D   E
  </div>

  <!-- Third Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    Graphs

    A ─── B
    │   ╱ │
    │ ╱   │
    C ─── D

  </div>
</div>

## A Pointer Does Not Keep its Target Alive
Lets make a generator function for `bob` `Persons`. In case there are multiple `Alices` out there that need friends:
```
Person* makeFriend() {
    Person bob;        // bob lives on the stack, only inside this function
    return &bob;       // hand out bob's address...
}                      // ...but bob is destroyed HERE, at the closing brace

Person* p = makeFriend();
p->age;                // 💥 dangling: p points at a bob that no longer exists
```

### Dangling Pointers
The pointer is still perfectly valid-looking. It holds a real address but the object at that address is gone. *A pointer does not keep its target alive.* Contrast this with what Java does: The GC keeps objects alive as long as any reference to the object exists.

## What is an Object's Lifetime?
What if I want bob to outlive the function that created him to avoid the dangling pointer above? That's precisely what the heap and `new` is for.
```
Person* makeFriend() {
    Person* bob = new Person;   // bob lives on the heap now
    return bob;                 // ✅ still valid after return — heap outlives scope
}
```

```
     Program's Memory (RAM)
   ┌───────────────────────────┐  high addresses
   │          STACK            │
   │  (local variables,        │
   │   function frames)        │
   │                           │
   │   grows downward ↓        │
   ├─────────────┬─────────────┤
   │             ▼             │
   │                           │
   │      (free space)         │
   │                           │
   │             ▲             │
   ├─────────────┴─────────────┤
   │   grows upward ↑          │
   │                           │
   │           HEAP            │
   │  (objects from `new`)     │
   ├───────────────────────────┤
   │   static / globals        │
   ├───────────────────────────┤
   │   code (your compiled     │
   │        instructions)      │
   └───────────────────────────┘  low addresses
```

```
    Person* alice = new Person;   // alice.age = 30

        STACK                          HEAP
   ┌──────────────┐              ┌──────────────┐
   │  alice       │              │   (Person)   │
   │  ┌────────┐  │              │  ┌────────┐  │
   │  │ 0x1A00 │──┼─────────────▶│  │ age:30 │  │  ← the actual Person
   │  └────────┘  │              │  └────────┘  │     lives at 0x1A00
   │   (a pointer,│              │              │
   │    8 bytes)  │              └──────────────┘
   └──────────────┘
     the handle                   the object

```

But... `new` doesn't clean itself up:

```
Person* p = makeFriend();
// ... use p ...
// forgot delete p;  →  memory leak: bob lives forever, unreachable
```
So `new` / `delete` solves the problem of lifetime, but uncovers the thorny question of **ownership**.

## Who is Responsible for an Object?
The ownership question is "who is responsible for calling delete, and when?" Here are all three failure modes with Alice and Bob; its genuinely difficult to keep track.

```
Person* alice = new Person;
Person* bob   = new Person;
alice->friend = bob;     // now TWO pointers reach bob: `bob` and `alice->friend`

delete bob;              // owner #1 frees it...
alice->friend->age;      // 💥 use-after-free: alice->friend now dangles
delete alice->friend;    // 💥 double-free if someone "cleans up" again
// or: nobody deletes → leak
```

The language doesn't track ownership for you. A raw pointer says nothing about who's responsible. The example above showed multiple pointers to one object but no overarching rule about who frees it. This leads to leaks, double-frees, or use-after-free gotchas.
The ambiguity THE problem, and it's a design problem.

## Conclusion
We have covered how pointers make modeling relationships between objects possible and also explored the fundamental pitfalls when working with pointers. These pitfalls point to fundamental questions of object lifetimes and ownership; the compiler has no answer for these.\
In the next post, we will cover modern C++'s answer to these questions.
